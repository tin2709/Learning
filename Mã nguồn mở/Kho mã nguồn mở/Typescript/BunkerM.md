Dựa trên toàn bộ mã nguồn và cấu trúc thư mục của dự án **BunkerM**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật lập trình và luồng hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án là một hệ thống "All-in-one" được đóng gói cực kỳ chặt chẽ:

*   **MQTT Broker:** **Eclipse Mosquitto** (hỗ trợ MQTT v3.1.1 và v5.0). Đây là trái tim của hệ thống.
*   **Backend (Đa dịch vụ):**
    *   **Python (FastAPI & Uvicorn):** Được dùng cho phần lớn các service chính như Monitor API, DynSec API (quản lý ACL), AWS/Azure Bridge API, và Smart Anomaly Detection.
    *   **Node.js (Express & Next.js):** Service `auth-api` để quản lý người dùng dashboard và Next.js cho giao diện quản trị mới (phiên bản v2).
*   **Frontend:**
    *   **Next.js 14 (React):** Sử dụng Tailwind CSS, Radix UI và Lucide icons (trong `Dockerfile.next`).
    *   **Vue.js (Vuetify & Ant Design):** Có một phiên bản frontend khác dựa trên Vue (có thể là bản cũ hoặc bản song song).
*   **Phân tích & AI:** **NumPy** và các thuật toán thống kê (Z-Score, EWMA) để phát hiện bất thường cục bộ.
*   **DevOps & Infrastructure:**
    *   **Docker & Docker Compose:** Đóng gói toàn bộ stack vào một Image duy nhất.
    *   **Nginx:** Làm Reverse Proxy điều phối traffic giữa giao diện web và các API backend.
    *   **Supervisord:** Quản lý vòng đời của tất cả các tiến trình (Mosquitto, nhiều API Python, Auth API) chạy song song bên trong một Container.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của BunkerM tuân theo tư duy **"Monolithic Container, Microservices Inside"**:

*   **Kiến trúc Đóng gói (Encapsulation):** Thay vì yêu cầu người dùng cài đặt Broker, DB, UI riêng lẻ, BunkerM gộp tất cả vào một thực thể duy nhất. Điều này giảm thiểu tối đa rào cản triển khai (chỉ cần 1 lệnh Docker).
*   **Sidecar-pattern nội bộ:** Các service Python chạy như những "công cụ hỗ trợ" (sidecars) cho Mosquitto. Ví dụ: `monitor-api` đọc dữ liệu từ broker, `dynsec-api` điều khiển quyền truy cập thông qua plugin `mosquitto_dynamic_security`.
*   **Cấu trúc dữ liệu phẳng (Flat Data Management):** Hệ thống ưu tiên sử dụng File JSON (`users.json`, `dynamic-security.json`) và SQLite cho các tính năng nhẹ để tránh việc phải cài đặt các hệ quản trị CSDL phức tạp như PostgreSQL hay MongoDB, giữ cho hệ thống gọn nhẹ.
*   **Tách biệt mối quan tâm (Separation of Concerns):** Dù chạy chung một container, mỗi tính năng (Bridge, Monitor, ACL, Anomaly) là một ứng dụng FastAPI riêng biệt, giúp việc bảo trì và mở rộng code-base dễ dàng hơn.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Lập trình bất đồng bộ (Asynchronous Programming):** Sử dụng triệt để `async/await` trong FastAPI và Next.js để xử lý đồng thời nhiều kết nối MQTT và yêu cầu API mà không gây nghẽn mạch.
*   **Cơ chế Proxy ngược (Server-side Proxying):** Next.js không gọi trực tiếp API backend từ Browser để tránh lộ API Key. Thay vào đó, nó tạo một Route Handler (`/api/proxy/[...path]`) để đính kèm API Key từ môi trường Server và forward yêu cầu đến các service Python nội bộ.
*   **Phân tích thống kê thực tế (Real-time Statistical Analysis):**
    *   Sử dụng **Sliding Windows** (cửa sổ trượt 1h và 24h) để tính toán Mean (trung bình) và StdDev (độ lệch chuẩn).
    *   Kỹ thuật phát hiện dựa trên ngưỡng Sigma (Z-Score) để lọc ra các giá trị MQTT vượt quá giới hạn thông thường.
*   **Quản lý tiến trình bằng Supervisord:** Sử dụng các file `.conf` để định nghĩa quyền ưu tiên (priority) khởi động. Ví dụ: Mosquitto khởi động trước, sau đó mới đến các API quản lý.
*   **Dynamic Configuration:** Kỹ thuật thao tác trực tiếp trên file cấu hình của Mosquitto và gửi tín hiệu `SIGHUP` hoặc restart qua `subprocess` để cập nhật cấu hình mà không làm gián đoạn hệ thống quá lâu.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Ingestion (Thu thập dữ liệu):
1.  `monitor-api` kết nối nội bộ đến Mosquitto qua giao thức MQTT.
2.  Nó đăng ký (subscribe) vào các topic hệ thống `$SYS/#` (để lấy stats) và tất cả topic `#` (để theo dõi lưu lượng).
3.  Dữ liệu được đẩy vào bộ nhớ đệm (Queue/Deque) và lưu xuống file JSON theo chu kỳ.

#### B. Luồng Phân tích bất thường (AI/Anomaly):
1.  `smart-anomaly` service định kỳ lấy dữ liệu từ `monitor-api`.
2.  Tính toán Baseline (ngưỡng cơ sở) cho từng topic.
3.  Nếu một message mới có giá trị lệch quá 3 lần độ lệch chuẩn (3-sigma), một `Anomaly` được ghi vào SQLite.
4.  Service `Alert` sẽ kiểm tra mức độ nghiêm trọng và tạo thông báo hiển thị lên Dashboard.

#### C. Luồng Quản lý bảo mật (ACL/Client):
1.  Người dùng thao tác trên giao diện (ví dụ: tạo Client mới).
2.  Frontend gửi request đến `dynsec-api`.
3.  `dynsec-api` thực thi lệnh `mosquitto_ctrl` (công cụ dòng lệnh của Mosquitto) để cập nhật plugin Dynamic Security.
4.  Plugin này tự động cập nhật file `dynamic-security.json`, Mosquitto áp dụng quyền mới ngay lập tức mà không cần khởi động lại.

#### D. Luồng Bridge (Kết nối đám mây):
1.  Người dùng cấu hình AWS IoT Core hoặc Azure IoT Hub qua UI.
2.  Backend tạo các file `.conf` trong thư mục `/etc/mosquitto/conf.d/`.
3.  Hệ thống ra lệnh restart Mosquitto để thiết lập kết nối cầu (Bridge) truyền dữ liệu từ local lên Cloud.

---

### Tóm tắt đặc điểm nổi bật
BunkerM không chỉ là một MQTT Broker; nó là một **Management Layer** thông minh. Sự kết hợp giữa **Nginx (Điều phối)** + **Mosquitto (Vận chuyển)** + **Python (Trí tuệ/Phân tích)** tạo ra một giải pháp IoT Edge mạnh mẽ, dễ triển khai nhưng vẫn đảm bảo khả năng can thiệp sâu vào bảo mật và hiệu suất.