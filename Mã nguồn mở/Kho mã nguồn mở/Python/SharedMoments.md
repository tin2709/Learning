thought Vietnamese.
Analyze a software project ("SharedMoments") based on its source code and structure, focusing on Core Technology, Architectural Thinking, Key Techniques, and Operational Flow.
A detailed README file in Vietnamese.

    *   *Backend:* Flask (Python 3.9), MySQL (Database), Python-Telegram-Bot, PyWebPush (Notifications).
    *   *Frontend:* HTML, CSS, JavaScript (Vanilla + jQuery), Materialize CSS (UI framework), Slick (Carousel), NanoGallery2.
    *   *DevOps:* Docker, Docker Compose.
    *   *Features:* Photo/Video feed, Gallery, Timeline (Moments), Countdown, Bucket/Movie list, Multi-language (Locales), Push Notifications (Telegram, Email, WebPush), Data Import/Export (CSV).

        *   Monolithic backend (Flask) organized by routes.
        *   Classic Client-Server (REST API).
        *   Database: Relational (MySQL).
        *   Security: Bcrypt for passwords, VAPID for web push.
        *   Session Management: Custom cookie-based session stored in DB.

        *   Localization (multi-language) system using JSON files.
        *   Background worker for scheduled tasks (cleanup, notifications).
        *   Docker-based deployment for easy setup.
        *   Dynamic side menu.
        *   API documentation (Swagger).

        1.  Docker start -> `docker_startup.sh` -> DB init.
        2.  Flask API start + Background Worker start.
        3.  Client (Frontend) requests -> `app.py` (middleware checks session).
        4.  Routes handle logic (Auth, Feed, etc.) via `DBController`.
        5.  Response returned as JSON.

    *   *Title:* SharedMoments - Nền tảng lưu giữ khoảnh khắc đôi lứa.
    *   *Section 1: Công nghệ cốt lõi (Core Technologies).* List Backend, Frontend, Database, DevOps.
    *   *Section 2: Tư duy kiến trúc (Architectural Design).* MVC-like approach, RESTful API, Service-oriented for notifications.
    *   *Section 3: Kỹ thuật then chốt (Key Technical Features).* Localization, Security, Notifications, Data management.
    *   *Section 4: Luồng hoạt động (Workflow).* Initialization -> Authentication -> Main Features.
    *   *Section 5: Đánh giá nhanh (Quick Review).* Simple, clean, effective for self-hosting.Dưới đây là bản phân tích chi tiết dự án **SharedMoments** dưới dạng file README tiếng Việt, tập trung vào các khía cạnh kỹ thuật và kiến trúc theo yêu cầu của bạn.

---

# SharedMoments - Tài liệu Phân tích Kỹ thuật

**SharedMoments** là một ứng dụng web tự lưu trữ (self-hosted) được thiết kế riêng cho các cặp đôi để lưu trữ kỷ niệm, quản lý danh sách mong muốn và theo dõi hành trình tình yêu.

## 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng các công nghệ phổ biến, ổn định, ưu tiên khả năng triển khai nhanh qua Docker:

*   **Backend:** 
    *   **Ngôn ngữ:** Python 3.9.
    *   **Framework:** Flask (Micro-framework) - linh hoạt và nhẹ nhàng cho các ứng dụng quy mô trung bình.
    *   **Xác thực:** Bcrypt (mã hóa mật khẩu) và hệ thống Session ID tùy chỉnh lưu trong cơ sở dữ liệu.
*   **Frontend:**
    *   **Giao diện:** HTML5, CSS3, Materialize CSS (Framework UI dựa trên Material Design).
    *   **Logic:** JavaScript (Vanilla JS & jQuery).
    *   **Thư viện hỗ trợ:** Slick (Carousel), nanogallery2 (Quản lý bộ sưu tập ảnh), lazysizes (Lazy loading để tối ưu hiệu suất hình ảnh).
*   **Cơ sở dữ liệu:**
    *   MySQL (thông qua Docker container riêng).
*   **Thông báo & Tương tác:**
    *   Telegram Bot API (Gửi thông báo qua Telegram).
    *   WebPush (Thông báo đẩy trên trình duyệt).
    *   SMTP (Gửi email thông báo).
*   **Triển khai:** 
    *   Docker & Docker Compose (Container hóa toàn bộ môi trường).

## 2. Tư duy kiến trúc (Architectural Thinking)

Dự án được xây dựng theo mô hình **Client-Server truyền thống** với sự tách biệt rõ ràng giữa logic và giao diện:

*   **Kiến trúc hướng API (API-First):** Backend hoạt động như một RESTful API cung cấp dữ liệu dưới dạng JSON. Giao diện người dùng (Frontend) gọi các API này để hiển thị dữ liệu.
*   **Module hóa Route:** Flask được chia nhỏ thành các blueprint/file route riêng biệt (ví dụ: `auth_routes.py`, `feed_routes.py`, `bucketlist_routes.py`) giúp dễ dàng quản lý và mở rộng tính năng mà không làm phình to file chính.
*   **Hệ thống Worker chạy ngầm (Background Worker):** Có một tiến trình chạy độc lập (`background_worker.py`) để xử lý các tác vụ định kỳ như gửi thông báo kỷ niệm hoặc dọn dẹp dữ liệu rác, giúp giảm tải cho server chính.
*   **Kiến trúc Stateless (một phần):** Session được lưu trong DB giúp việc kiểm tra trạng thái đăng nhập đồng nhất giữa các thiết bị.

## 3. Các kỹ thuật then chốt (Key Technical Features)

*   **Đa ngôn ngữ (Localization):** Sử dụng các file JSON trong thư mục `/locales`. Hệ thống tự động nhận diện ngôn ngữ qua biến môi trường `LOCALE` và ánh xạ text tương ứng từ Backend đến Frontend.
*   **Xử lý tệp tin (File Management):**
    *   Lưu trữ trực tiếp trên hệ thống tệp tin (không dùng S3/Cloud).
    *   Kỹ thuật dọn dẹp (Cleanup): Hệ thống tự đối chiếu file trong thư mục `upload` với dữ liệu trong DB để xóa các file không còn sử dụng.
*   **Bảo mật:**
    *   Middleware (`@app.before_request`): Kiểm tra Session ID cho mọi yêu cầu (trừ các trang công khai).
    *   Cơ chế Salted Hash: Kết hợp mã Hex ngẫu nhiên (salt) với mật khẩu trước khi băm bằng Bcrypt để chống lại tấn công Rainbow Table.
*   **Hệ thống thông báo đa kênh:** Hỗ trợ cùng lúc 3 kênh (WebPush, Telegram, Email) thông qua các lớp tiện ích (Utils).
*   **Import/Export CSV:** Cung cấp khả năng sao lưu và phục hồi toàn bộ dữ liệu (người dùng, cài đặt, feed) thông qua tệp CSV, giúp người dùng không bị phụ thuộc vào một server duy nhất.

## 4. Tóm tắt luồng hoạt động (Operational Flow)

### Luồng khởi tạo (Startup Flow)
1.  **Docker Compose** kích hoạt hai container: `sharedmoments` (Flask) và `sharedmoments-db` (MySQL).
2.  Script `docker_startup.sh` chạy:
    *   Đợi DB sẵn sàng.
    *   Chạy `db_init.py` để tạo bảng và dữ liệu mẫu nếu chưa có (thông qua SQL dump).
    *   Tạo cặp khóa VAPID (Private/Public Key) cho thông báo đẩy nếu chưa tồn tại.
    *   Khởi chạy song song Flask Server và Background Worker.

### Luồng xử lý yêu cầu (Request Flow)
1.  **Người dùng truy cập:** Trình duyệt tải các file tĩnh (HTML/JS) từ Flask.
2.  **Kiểm tra Session:** Mỗi khi gọi API, Middleware kiểm tra Cookie `session_id`.
    *   Nếu hợp lệ: Trả về dữ liệu.
    *   Nếu không: Trả về lỗi 401 (Unauthorized), Frontend chuyển hướng về trang Login.
3.  **Tương tác dữ liệu:** 
    *   Frontend gọi API (ví dụ: `/api/v1/feed`).
    *   Backend gọi `DBController` để truy vấn MySQL.
    *   Dữ liệu được trả về và JS Render vào DOM của Materialize CSS.

### Luồng tác vụ ngầm (Worker Flow)
1.  Worker tính toán thời gian ngủ đến mốc giờ được cấu hình (mặc định 6:00 sáng).
2.  Khi thức dậy:
    *   Dọn dẹp các session hết hạn.
    *   Kiểm tra xem hôm nay có phải ngày kỷ niệm không (tháng, nửa năm, năm).
    *   Nếu có, gửi thông báo qua tất cả các kênh đã cấu hình.

## 5. Kết luận
SharedMoments là một ví dụ tốt về việc ứng dụng Flask để giải quyết nhu cầu cá nhân một cách bài bản. Dự án không quá phức tạp về mặt thuật toán nhưng rất chỉn chu về mặt **trải nghiệm người dùng (UX)**, **khả năng đóng gói (Docker)** và **tính riêng tư (Self-hosted)**.