Chào bạn, đây là bản phân tích chuyên sâu về dự án **Anthias** (tiền thân là Screenly OSE) - giải pháp biển báo kỹ thuật số (Digital Signage) mã nguồn mở phổ biến nhất cho Raspberry Pi và PC.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Anthias sử dụng một tập hợp công nghệ đa dạng, được tối ưu hóa cho các thiết bị nhúng (Edge Devices):

*   **Backend:** **Django (Python 3.11+)**. Dự án đã chuyển đổi từ Flask sang Django để tận dụng hệ sinh thái quản lý dữ liệu và bảo mật mạnh mẽ hơn.
*   **Frontend:** **React + TypeScript + Redux Toolkit**. Đây là bước hiện đại hóa UI, giúp giao diện quản lý mượt mà và dễ bảo trì.
*   **Containerization:** **Docker & Docker Compose**. Toàn bộ hệ thống chạy dưới dạng các microservices trong container, giúp triển khai nhất quán trên nhiều dòng Raspberry Pi (từ Pi 1 đến Pi 5) và x86.
*   **Giao tiếp thời gian thực:** 
    *   **ZeroMQ (ZMQ):** Dùng để truyền tin nhắn tốc độ cao, độ trễ thấp giữa Server và Viewer.
    *   **WebSockets:** Cung cấp cập nhật trạng thái thời gian thực lên giao diện người dùng.
*   **Xử lý tác vụ ẩn:** **Celery + Redis**. Xử lý việc tải tài nguyên (assets), dọn dẹp hệ thống và các tác vụ nặng không đồng bộ.
*   **Trình duyệt hiển thị (WebView):** **C++ & Qt (Qt5/Qt6)**. Một trình duyệt tùy chỉnh được xây dựng riêng để render nội dung lên màn hình với hiệu suất phần cứng tối ưu.
*   **Cơ sở dữ liệu:** **SQLite**. Phù hợp cho lưu trữ cục bộ trên thẻ nhớ SD của Raspberry Pi.
*   **Cấu hình hệ thống:** **Ansible**. Dùng để tự động hóa việc thiết lập hệ điều hành, mạng và các unit systemd.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án Anthias được thiết kế theo kiến trúc **Distributed Microservices trên Edge**:

*   **Tách biệt mặt phẳng điều khiển (Control Plane) và mặt phẳng hiển thị (Data Plane):**
    *   `anthias-server` đóng vai trò điều khiển, quản lý API và lịch chiếu.
    *   `anthias-viewer` và `webview` đóng vai trò thực thi, chỉ quan tâm đến việc hiển thị cái gì lên màn hình.
*   **Đa phiên bản API (API Versioning):** Trong thư mục `api/`, hệ thống duy trì từ v1 đến v2. Điều này cực kỳ quan trọng cho các thiết bị IoT vì người dùng có thể không cập nhật phần mềm đồng thời, cần đảm bảo tính tương thích ngược.
*   **Thiết kế cho sự ổn định (Resilience):** Sử dụng các cơ chế như "watchdog" và các task định kỳ của Celery để đảm bảo nếu một thành phần treo, hệ thống vẫn có khả năng tự phục hồi hoặc dọn dẹp tài nguyên thừa.
*   **Hỗ trợ đa nền tảng (Hardware Abstraction):** Thông qua Docker và Jinja2 templates cho Dockerfile, dự án tách biệt logic cài đặt cho từng loại board (pi1, pi4, pi5, x86) nhưng vẫn giữ chung một mã nguồn ứng dụng.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Jinja2 Templating cho Docker:** Thay vì viết hàng chục Dockerfile cố định, Anthias dùng template `.j2` để phát sinh Dockerfile linh hoạt dựa trên cấu trúc phần cứng của thiết bị mục tiêu.
*   **Mixin Pattern trong API:** Sử dụng các lớp Mixin (`DeleteAssetViewMixin`, `BackupViewMixin`) trong Django Rest Framework để tái sử dụng logic xử lý giữa các phiên bản API khác nhau mà không làm lặp mã.
*   **ZMQ Publisher/Subscriber:** Kỹ thuật này cho phép Server gửi lệnh "reload" hoặc "skip asset" tới Viewer ngay lập tức mà không cần Viewer phải liên tục gửi request kiểm tra (polling).
*   **TypeScript Strict Typing:** Dự án áp dụng quy tắc nghiêm ngặt (định nghĩa trong `.cursor/rules`), cấm sử dụng kiểu `any` để đảm bảo an toàn về kiểu dữ liệu trong ứng dụng frontend phức tạp.
*   **Sử dụng `uv` cho Python:** Thay thế Poetry/Pip truyền thống bằng `uv` để quản lý dependencies nhanh hơn, phù hợp với môi trường CI/CD và cài đặt trên thiết bị yếu.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Quản lý (Management):**
    *   Người dùng truy cập Web UI -> React gửi yêu cầu tới Django API.
    *   Tài nguyên (Ảnh/Video/URL) được lưu vào SQLite và file vật lý được tải về thông qua Celery.
2.  **Giai đoạn Lập lịch (Scheduling):**
    *   `viewer/scheduling.py` liên tục tính toán dựa trên thời gian thực tế để quyết định asset nào cần được phát tiếp theo.
3.  **Giai đoạn Truyền tin (Communication):**
    *   Server gửi một thông điệp qua cổng ZMQ (thường là 10001).
    *   `anthias-viewer` (Subscriber) nhận được thông điệp này.
4.  **Giai đoạn Hiển thị (Rendering):**
    *   Nếu là video, Viewer gọi trình phát media (OMXPlayer hoặc VLC).
    *   Nếu là ảnh hoặc trang web, Viewer ra lệnh cho `webview` (Qt) tải URL tương ứng.
    *   NGINX đóng vai trò proxy ngược để phục vụ các asset tĩnh và định tuyến yêu cầu WebSocket.

---

### 5. Đánh giá tổng quan

**Anthias** là một dự án mã nguồn mở có độ hoàn thiện rất cao về mặt kỹ thuật. Sự kết hợp giữa **Python/Django** (linh hoạt) và **C++/Qt** (hiệu năng hiển thị) cùng với việc đóng gói bằng **Docker** tạo nên một giải pháp cực kỳ mạnh mẽ cho lĩnh vực Digital Signage.

Điểm sáng nhất của dự án là khả năng chuyển mình từ một mã nguồn cũ (Legacy) sang các công nghệ hiện đại (React/TypeScript/Django) mà vẫn giữ được cộng đồng hỗ trợ đông đảo và khả năng chạy trên các phần cứng cực yếu như Raspberry Pi Zero.