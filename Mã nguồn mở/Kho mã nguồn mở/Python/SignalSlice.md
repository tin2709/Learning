Dựa trên mã nguồn và tài liệu của dự án **SignalSlice**, dưới đây là phân tích chi tiết về hệ thống giám sát "Chỉ số Pizza Lầu Năm Góc" (Pentagon Pizza Index) theo các khía cạnh kỹ thuật:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án sử dụng một tập hợp công nghệ hiện đại, tập trung vào việc thu thập dữ liệu tự động và truyền tải thời gian thực:

*   **Backend:** **Python 3.12+** kết hợp với **Flask**. Đây là trung tâm điều phối các dịch vụ và cung cấp API.
*   **Real-time Communication:** **Flask-SocketIO (WebSocket)**. Đây là thành phần quan trọng nhất giúp đẩy các cập nhật từ scanner (như trạng thái scraping, phát hiện bất thường) trực tiếp lên giao diện người dùng mà không cần tải lại trang.
*   **Web Scraping & Automation:** **Playwright**. Được sử dụng để giả lập trình duyệt, truy cập Google Maps và trích xuất dữ liệu "Popular Times" (thời gian phổ biến) của các nhà hàng pizza.
*   **Frontend:**
    *   **JavaScript (ES6+):** Xử lý logic hiển thị và kết nối Socket.IO.
    *   **Chart.js:** Hiển thị biểu đồ xu hướng và các đồng hồ đo (gauges) chỉ số.
    *   **Leaflet:** Bản đồ tương tác hiển thị vùng giám sát và mức độ đe dọa (threat levels).
    *   **CSS3:** Thiết kế theo phong cách giao diện giám sát quân sự (Surveillance UI) với hiệu ứng animation radar.
*   **Data & Storage:** **CSV & JSON**. Hệ thống sử dụng file CSV để lưu trữ dữ liệu lịch sử và `baseline.json` để định nghĩa các ngưỡng hoạt động bình thường.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án đang trong quá trình chuyển đổi từ kiến trúc nguyên khối (Monolithic) sang **Kiến trúc phân lớp (Layered Architecture)** để tăng khả năng bảo trì:

*   **Service Layer (`services/scanner_service.py`):** Tách biệt logic nghiệp vụ của việc quét dữ liệu ra khỏi các route của Flask.
*   **State Management (`state_manager.py`):** Sử dụng mô hình **Singleton** và **Thread-safe**. Mọi trạng thái của ứng dụng (chỉ số pizza, số lượng anomaly) được quản lý tập trung và bảo vệ bằng `threading.Lock()` để tránh xung đột dữ liệu khi nhiều tiến trình chạy song song.
*   **Observer Pattern:** Khi trạng thái trong `StateManager` thay đổi, nó tự động thông báo để SocketIO phát tín hiệu tới các client đang kết nối.
*   **Adapter Pattern (`scraping/scraper_adapter.py`):** Cho phép hệ thống cũ vẫn hoạt động trong khi đang nâng cấp lên các module scraper mới hiệu quả hơn.

---

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **Chiến lược Scraping thông minh:** Thay vì dùng API đắt đỏ, hệ thống sử dụng Playwright để quét các nhãn `aria-label` trên Google Maps. Nó ưu tiên dữ liệu **"LIVE"** (đang bận X% ngay lúc này) hơn là dữ liệu dự đoán lịch sử.
*   **Thuật toán phát hiện bất thường (Anomaly Detection):**
    *   Tính toán trung bình trượt (rolling averages) dựa trên ngày trong tuần và giờ trong ngày.
    *   Sử dụng ngưỡng cấu hình (mặc định là **25%**) trên mức cơ sở (baseline) để kích hoạt cảnh báo.
*   **Xử lý bất đồng bộ (Asynchronous Processing):** Sử dụng `asyncio` để quản lý các tác vụ scraping tốn thời gian mà không làm treo server Flask.
*   **Data Validation & Sanitization:** Module `validation.py` thực hiện kiểm tra nghiêm ngặt dữ liệu đầu vào từ web (kiểu dữ liệu, định dạng URL, giá trị phần trăm 0-100) để đảm bảo tính toàn vẹn của dữ liệu OSINT.

---

### 4. Luồng hoạt động hệ thống (Operational Workflow)

Quy trình hoạt động của SignalSlice diễn ra theo một vòng lặp khép kín:

1.  **Kích hoạt (Trigger):** Bộ lập lịch (`scheduler.py`) kích hoạt mỗi giờ một lần hoặc người dùng nhấn nút "Trigger Scan" trên Dashboard.
2.  **Thu thập (Collection):**
    *   Playwright khởi động trình duyệt ngầm (headless).
    *   Truy cập danh sách URL các nhà hàng trong bán kính 50 dặm quanh Lầu Năm Góc.
    *   Trích xuất dữ liệu bận rộn hiện tại.
3.  **Phân tích (Analysis):**
    *   Dữ liệu mới được nạp vào `anomalyDetect.py`.
    *   Hệ thống so sánh chỉ số hiện tại với `baseline.json`.
    *   Nếu vượt ngưỡng, đánh dấu là một **"Live Anomaly"**.
4.  **Cập nhật trạng thái (State Update):**
    *   `StateManager` cập nhật "Chỉ số Pizza" (tăng lên nếu bận rộn) và "Chỉ số Bar Gay" (thường có xu hướng ngược lại trong OSINT thực tế).
    *   Ghi nhật ký vào feed hoạt động.
5.  **Phát sóng (Broadcasting):**
    *   SocketIO gửi các sự kiện `activity_update` và `pizza_index_update` tới frontend.
6.  **Hiển thị (Visualization):**
    *   Dashboard thay đổi màu sắc (sang đỏ nếu có anomaly).
    *   Biểu đồ Chart.js vẽ lại đường xu hướng.
    *   Bản đồ Leaflet cập nhật mức độ đe dọa tại các điểm nóng.

### Tổng kết
SignalSlice là một dự án thú vị kết hợp giữa kỹ thuật **OSINT** (tình báo nguồn mở) và phát triển web hiện đại. Nó minh chứng cho việc biến những dữ liệu công cộng dường như vô hại (độ bận rộn của quán ăn) thành các chỉ số có giá trị phân tích chiến lược thông qua tự động hóa và phân tích thống kê.