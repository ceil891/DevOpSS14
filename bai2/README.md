# Bài 2: Vận dụng cơ bản - "Vẽ biểu đồ sức khỏe VPS"

## 1. Mục tiêu kiến thức
- Viết các câu truy vấn PromQL cơ bản để tính toán hiệu năng hệ thống từ Node Exporter.
- Hiểu và phân biệt các loại biểu đồ trực quan trong Grafana: **Time series** (chuỗi thời gian), **Stat** (con số thống kê tổng hợp), **Gauge** (đồng hồ đo áp suất).
- Sử dụng tính năng **Standard Options > Unit** để hiển thị đơn vị dữ liệu chuẩn xác mà không can thiệp méo mó vào câu truy vấn.

---

## 2. Tình huống nghiệp vụ
Ban lãnh đạo (Sếp) cần một màn hình giám sát tinh gọn, gồm 2 chỉ số sống còn của máy chủ VPS:
1. **Mức sử dụng CPU hiện tại (%)**: Dưới dạng biểu đồ đường theo thời gian thực để nhận diện các đợt tải đột biến (Spike).
2. **Lượng RAM còn trống**: Dưới dạng một con số to bản (Stat Panel) nổi bật để người quản lý dễ dàng nhìn thấy và ước tính kế hoạch nâng cấp hạ tầng.

---

## 3. Hướng dẫn tạo Dashboard và xây dựng 2 biểu đồ

### Bước 1: Tạo Dashboard mới
1. Đăng nhập vào Grafana (`http://localhost:3000`).
2. Nhấn menu dấu cộng **(+)** ở góc trên bên phải hoặc vào **Dashboards** > **New** > **New dashboard**.
3. Nhấn nút **+ Add visualization**.
4. Chọn Data Source là **Prometheus**.

---

### Biểu đồ 1: Biểu đồ Time Series đo % CPU của máy chủ

#### 1. Câu truy vấn PromQL
Tại ô truy vấn `Query (PromQL)`, nhập:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

#### 2. Giải phẫu chi tiết câu truy vấn PromQL
- `node_cpu_seconds_total`: Metric dạng Counter lưu tổng số giây CPU đã hoạt động kể từ khi máy khởi động, phân loại theo nhãn `mode` (`idle`, `user`, `system`, `iowait`, ...).
- `{mode="idle"}`: Bộ lọc chỉ lấy thời gian CPU ở trạng thái nghỉ (rảnh rỗi).
- `rate(...[5m])`: Hàm tính tốc độ tăng của counter trên cửa sổ thời gian 5 phút. Kết quả trả về tỷ lệ thời gian idle từ `0` đến `1` cho từng core CPU.
- `avg by (instance) (...)`: Tính giá trị trung bình trên tất cả các core CPU của server (tránh trường hợp máy có 4 core hiển thị 4 đường riêng biệt).
- `* 100`: Quy đổi từ tỷ lệ `[0, 1]` sang phần trăm `[0, 100]%` thời gian rảnh.
- `100 - (...)`: Công thức bù nghịch đảo: Lấy 100% trừ đi % thời gian rảnh để ra **% CPU thực tế đang bận sử dụng**.

#### 3. Cấu hình giao diện Panel (Thanh bên phải)
- **Visualization**: Chọn **Time series**.
- **Panel options**:
  - Title: `CPU Usage (%)`
- **Standard options**:
  - **Unit**: Chọn `Misc` > `Percent (0-100)` (hoặc gõ tìm kiếm `percent`).
  - **Min**: `0`
  - **Max**: `100`
- **Graph styles**:
  - Line width: `2`
  - Fill opacity: `15%` hoặc `20%` (tạo nền bóng đẹp bên dưới đường cong).
- Nhấn **Apply** (góc trên bên phải).

---

### Biểu đồ 2: Biểu đồ Stat đo lượng RAM còn trống

Nhấn **+ Add visualization** một lần nữa trên dashboard để tạo panel thứ hai:

#### 1. Câu truy vấn PromQL
Tại ô `Query (PromQL)`, nhập:

```promql
node_memory_MemFree_bytes
```

*(Lưu ý: Nếu có nhiều server, bạn có thể lọc theo instance: `node_memory_MemFree_bytes{instance="localhost:9100"}` hoặc để PromQL tự hiển thị).*

#### 2. Cấu hình giao diện Panel (Thanh bên phải)
- **Visualization**: Chọn **Stat**.
- **Panel options**:
  - Title: `Free RAM`
- **Value options**:
  - **Show**: `Calculate`
  - **Calculation**: Chọn `Last *` (hoặc `LastNotNull` - tức lấy giá trị đo đạc mới nhất hiện tại).
- **Stat styles**:
  - Color mode: `Value` hoặc `Background`
  - Graph mode: `Area` (để hiển thị một biểu đồ thu nhỏ dạng tia sparkline phía sau con số).
- **Thresholds (Ngưỡng màu cảnh báo trực quan)**:
  - Base: `Red` (RAM quá thấp)
  - `536870912` (512MB): `Orange`
  - `1073741824` (1GB): `Green` (RAM dồi dào an toàn)

---

## 4. Giải thích Bẫy: Xử lý hiển thị đơn vị Bytes sang MB/GB

### Vấn đề:
Truy vấn `node_memory_MemFree_bytes` trả về một con số thô khổng lồ tính bằng Bytes (ví dụ: `2147483648` Bytes).
Nhiều người thường có thói quen chia tay trong câu truy vấn:
```promql
# Cách làm SAI hoặc KÉM TỐI ƯU:
node_memory_MemFree_bytes / 1024 / 1024
```

### Vì sao không nên chia thủ công trong câu truy vấn?
1. **Mất khả năng thích ứng linh hoạt (Auto-scaling)**: Nếu máy chủ có 32GB RAM, biểu đồ chia `/ 1024 / 1024` sẽ hiện `32768 MB` rất dài và khó đọc; nếu máy server lớn có 256GB thì con số còn rối rắm hơn.
2. **Sai lệch dữ liệu gốc**: Việc chia trực tiếp trong PromQL biến đổi bản chất metric gốc thành một số vô hướng không có siêu dữ liệu đơn vị đi kèm.
3. **Mất tính năng định dạng thông minh của Grafana**: Khi viết `/ 1024 / 1024`, Grafana chỉ coi đó là một con số vô danh (Number), không tự chèn thêm hậu tố `MB` hay `GB`.

---

### Giải pháp chuẩn xác trong Grafana:
**Sử dụng tính năng Unit Formatting ở thanh bên phải của Panel:**

1. Chuyển sang thanh **Panel edit** ở cạnh phải màn hình.
2. Tìm đến mục **Standard options** > trường **Unit**.
3. Nhấp vào ô tìm kiếm Unit và chọn:
   ```text
   Data > bytes (IEC)
   ```
   *(Hoặc gõ chữ `bytes` vào ô tìm kiếm và chọn `bytes(IEC)`).*

![Grafana Unit Config](https://grafana.com/docs/grafana/latest/static/img/docs/panels-visualizations/standard-options-unit.png)

#### Cơ chế hoạt động của `bytes (IEC)`:
- Grafana sẽ tự động tính toán theo bội số của `1024` ($2^{10}$):
  - Khi RAM còn `500,000,000` Bytes $\rightarrow$ Tự động hiển thị `476.8 MiB`.
  - Khi RAM còn `4,000,000,000` Bytes $\rightarrow$ Tự động hiển thị `3.72 GiB`.
  - Khi RAM vượt mức Terabyte $\rightarrow$ Tự động nhảy sang `TiB`.
- Nếu sếp yêu cầu bắt buộc chỉ hiện đơn vị Megabytes, bạn có thể chọn thẳng `Data > megabytes (Mbyte / MiB)`.
- Nhờ vậy, câu PromQL giữ nguyên bản thể chuẩn: `node_memory_MemFree_bytes`.

---

## 5. File Dashboard JSON đính kèm
Để nhập nhanh dashboard mẫu đã dựng sẵn 2 panel trên:
- Sử dụng file: [dashboard-vps.json](file:///E:/BaiTapDeVop/SS14/bai2/dashboard-vps.json)
- Thao tác: **Dashboards** > **New** > **Import** > Tải file `dashboard-vps.json` lên.
