Dựa trên mã nguồn và tài liệu của kho lưu trữ `evcc`, dưới đây là phân tích chi tiết về dự án theo yêu cầu của bạn:

---

### 1. Công nghệ cốt lõi (Core Technologies)

`evcc` là một ứng dụng hiện đại kết hợp giữa hiệu suất cao của Go và tính linh hoạt của Vue.js:

*   **Backend (Go):** 
    *   Sử dụng **Go (Golang)** làm ngôn ngữ chính để xử lý logic điều khiển thời gian thực và giao tiếp thiết bị.
    *   **GORM & SQLite:** Quản lý cơ sở dữ liệu cho các phiên sạc (sessions) và cấu hình.
    *   **Giao thức truyền thông:** Hỗ trợ đa dạng các giao thức công nghiệp như **Modbus (TCP/RTU)**, **MQTT**, **HTTP/REST**, **WebSockets**, **gRPC**, **EEBus** và **OCPP**.
    *   **Code Generation:** Sử dụng `go generate` rộng rãi để tạo các mã trang trí (decorators), mocks để test và các enum.
*   **Frontend (Vue.js & TypeScript):**
    *   **Vue 3 (Options API):** Xây dựng giao diện người dùng (UI).
    *   **Vite:** Công cụ build frontend cực nhanh.
    *   **Tailwind/Bootstrap CSS:** Tùy chỉnh giao diện với CSS Variables để hỗ trợ Dark Mode.
    *   **Chart.js:** Hiển thị biểu đồ năng lượng và dữ liệu thời gian thực.
*   **Hạ tầng & Deployment:**
    *   **Docker:** Hỗ trợ đa nền tảng (amd64, arm64, armv6).
    *   **gokrazy:** Hỗ trợ chạy trực tiếp trên các thiết bị nhúng như Raspberry Pi mà không cần OS phức tạp.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của `evcc` được thiết kế theo hướng **Modularity (Tính module)** và **Vendor-agnostic (Không phụ thuộc nhà cung cấp)**:

*   **Plugin-based Architecture:** Mỗi loại thiết bị (Charger, Meter, Vehicle) được xem như một Plugin. Điều này cho phép mở rộng hỗ trợ hàng trăm thiết bị khác nhau chỉ bằng cách thêm file cấu hình YAML (Templates) hoặc mã nguồn Go mới mà không làm ảnh hưởng đến lõi hệ thống.
*   **Local-first & Privacy:** Hệ thống ưu tiên quản lý năng lượng tại chỗ (Local), giảm thiểu sự phụ thuộc vào Cloud của các hãng xe/trạm sạc, tăng tốc độ phản hồi và bảo mật dữ liệu.
*   **Interface-driven Development:** Sử dụng các `Interface` trong Go (như `api.Meter`, `api.Charger`) để định nghĩa hành vi. Bất kể thiết bị sạc của hãng nào, chỉ cần tuân thủ Interface là có thể tích hợp vào hệ thống.
*   **State Management:** Một vòng lặp điều khiển (Control Loop) trung tâm liên tục thu thập dữ liệu từ các cảm biến năng lượng (Meters) và đưa ra quyết định sạc dựa trên thuật toán tối ưu hóa (Optimizer).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Decorator Pattern:** Dự án sử dụng kỹ thuật này (trong các file `*_decorators.go`) để thêm các tính năng bổ sung cho thiết bị (ví dụ: một bộ sạc cơ bản được "trang trí" thêm tính năng đo năng lượng hoặc chuyển đổi pha sạc).
*   **Template Engine:** Hệ thống cấu hình dựa trên file YAML mạnh mẽ (`templates/definition/`), giúp người dùng cấu hình thiết bị phức tạp chỉ thông qua vài tham số đơn giản.
*   **Embedded Assets:** Sử dụng `go:embed` để nén toàn bộ mã nguồn frontend (HTML/JS/CSS) vào trong một file nhị phân Go duy nhất, giúp việc cài đặt và phân phối cực kỳ dễ dàng.
*   **Smart Charging Algorithms:** Thuật toán tự động điều chỉnh dòng điện sạc dựa trên:
    *   Dư lượng điện mặt trời (PV excess).
    *   Giá điện theo thời gian thực (Dynamic pricing từ các nhà cung cấp như Tibber, AWATTAR).
    *   Kế hoạch sạc (Charging plans) do người dùng thiết lập.
*   **Real-time Updates:** Kết hợp WebSockets và MQTT để cập nhật trạng thái sạc lên giao diện người dùng ngay lập tức mà không cần tải lại trang.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow)

Bạn có thể hình dung luồng hoạt động của `evcc` như một "nhạc trưởng" điều phối năng lượng trong nhà:

#### Bước 1: Thu thập dữ liệu (Input)
Hệ thống liên tục đọc dữ liệu từ:
*   **Grid Meter:** Lượng điện đang mua/bán với lưới điện.
*   **PV Inverter:** Lượng điện mặt trời đang sản xuất.
*   **Battery System:** Trạng thái pin lưu trữ gia đình.
*   **Vehicle (EV):** Trạng thái pin (SoC) và yêu cầu sạc của xe.

#### Bước 2: Phân tích và Tối ưu hóa (Process)
Bộ não trung tâm (`site.go` và `loadpoint.go`) sẽ tính toán:
*   "Hiện tại tôi có bao nhiêu điện dư?"
*   "Xe có cần sạc gấp không (dựa trên kế hoạch sạc)?"
*   "Giá điện hiện tại có đang rẻ không?"
*   **Optimizer** sẽ quyết định dòng điện sạc tối ưu (ví dụ: 6A, 10A hoặc 16A).

#### Bước 3: Điều khiển thực thi (Output)
Hệ thống gửi lệnh đến:
*   **Wallbox/Charger:** Bật/tắt sạc hoặc điều chỉnh giới hạn dòng điện.
*   **Smart Switches:** Bật/tắt các thiết bị tiêu thụ khác (như máy bơm, bình nóng lạnh) để tận dụng điện dư.

#### Bước 4: Giám sát và Thông báo (Feedback)
*   Dữ liệu được lưu vào SQLite/InfluxDB để báo cáo.
*   Gửi thông báo qua Telegram/Pushover khi xe bắt đầu sạc hoặc hoàn thành.
*   UI hiển thị biểu đồ dòng năng lượng sống động cho người dùng theo dõi.

---
**Kết luận:** `evcc` là một dự án mã nguồn mở cực kỳ chuyên nghiệp, giải quyết bài toán khó trong việc chuẩn hóa giao tiếp giữa hàng trăm loại thiết bị năng lượng khác nhau để đạt được mục tiêu "sạc xe bằng ánh nắng mặt trời" một cách hiệu quả nhất.