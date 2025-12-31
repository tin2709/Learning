Dưới đây là bản tóm tắt phân tích dự án **listmonk** (một hệ thống quản lý newsletter và mailing list mã nguồn mở nổi tiếng) tập trung vào công nghệ, kiến trúc và luồng hoạt động, trình bày dưới dạng tệp README tiếng Việt.

---

# Phân Tích Hệ Thống listmonk (Newsletter & Mailing List Manager)

Dự án **listmonk** là một ứng dụng self-hosted hiệu suất cao, được đóng gói trong một file thực thi duy nhất (single binary). Dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống.

## 1. Công Nghệ Cốt Lõi (Core Technologies)

### Backend (Ngôn ngữ Go)
*   **Echo Framework:** Framework web tốc độ cao để xây dựng RESTful API.
*   **Sqlx:** Mở rộng cho thư viện `database/sql` của Go, giúp tương tác với PostgreSQL dễ dàng và hiệu quả hơn.
*   **Koanf:** Thư viện quản lý cấu hình (config) mạnh mẽ, hỗ trợ nhiều định dạng (TOML, JSON, ENV).
*   **Stuffbin:** Công nghệ đóng gói tất cả tài nguyên tĩnh (HTML, CSS, JS, SQL) vào trong file binary Go.
*   **Go-routines & Channels:** Tận dụng tối đa khả năng xử lý song song để gửi hàng triệu email đồng thời mà không nghẽn hệ thống.

### Frontend (Vue.js)
*   **Vue.js 3:** Xây dựng dashboard hiện đại, phản ứng nhanh.
*   **Buefy / Bulma:** CSS Framework nhẹ nhàng, dễ tùy chỉnh cho UI.
*   **TinyMCE & Email Builder:** Bộ soạn thảo trực quan (WYSIWYG) và trình kéo thả để thiết kế mẫu email chuyên nghiệp.

### Cơ sở dữ liệu (PostgreSQL)
*   Sử dụng các tính năng nâng cao như **JSONB** để lưu trữ thuộc tính tùy chỉnh của người đăng ký (attribs).
*   **Materialized Views:** Tăng tốc độ hiển thị báo cáo (stats) trên Dashboard mà không cần tính toán lại từ đầu các truy vấn phức tạp.
*   **ENUM types:** Đảm bảo tính toàn vẹn dữ liệu cho các trạng thái (Campaign status, Subscription status).

---

## 2. Tư Duy Kiến Trúc (Architectural Thinking)

Hệ thống được thiết kế theo hướng **Modular Monolith** (Nguyên khối nhưng phân rã theo module nội bộ):

*   **Single Binary Philosophy:** Mục tiêu là dễ dàng triển khai. Chỉ cần 1 file thực thi và 1 cơ sở dữ liệu Postgres là hệ thống có thể chạy ngay lập tức.
*   **Stateless Backend:** Backend xử lý logic mà không lưu trạng thái trên đĩa cục bộ (trừ phần upload file nếu không dùng S3), giúp dễ dàng mở rộng.
*   **Worker-based Manager:** Tách biệt luồng xử lý Web API và luồng xử lý gửi email (Campaign Manager).
*   **Plugin-friendly Messengers:** Kiến trúc cho phép dễ dàng thêm các phương thức gửi tin nhắn mới (SMTP, SES, SendGrid hoặc các HTTP Postback cho SMS/Messenger).

---

## 3. Các Kỹ Thuật Chính (Key Techniques)

### Hiệu suất gửi email (High Throughput)
*   **Batching:** Thay vì truy vấn từng email, hệ thống lấy dữ liệu theo từng lô (batches) để tối ưu hóa IO cơ sở dữ liệu.
*   **Sliding Window Rate Limiting:** Kiểm soát tốc độ gửi email linh hoạt để tránh bị các nhà cung cấp (như Gmail/Amazon SES) đánh dấu là spam.

### Quản lý người dùng và Bảo mật
*   **RBAC (Role-Based Access Control):** Phân quyền chi tiết (ví dụ: người chỉ xem báo cáo, người có quyền gửi campaign).
*   **OIDC & 2FA:** Hỗ trợ đăng nhập qua Google/GitHub và bảo mật 2 lớp (TOTP).
*   **Altcha (Captcha bảo mật):** Giải pháp Captcha hiện đại, không cần cookie, bảo vệ form đăng ký công cộng khỏi bot.

### Hệ thống Tracking (Theo dõi)
*   **Tracking Pixel:** Chèn một ảnh 1x1 trong suốt vào cuối mỗi email để ghi nhận lượt mở (Open rate).
*   **Link Wrapping:** Mã hóa các liên kết trong nội dung email để theo dõi lượt click của từng cá nhân mà vẫn đảm bảo tính riêng tư nếu được cấu hình.

---

## 4. Tóm Tắt Luồng Hoạt Động (Workflow)

### Bước 1: Khởi tạo (Initialization)
1.  **Load Config:** Đọc file `config.toml` và các biến môi trường (ENV).
2.  **DB Migration:** Tự động kiểm tra và nâng cấp cấu hình bảng trong Postgres (`--upgrade`).
3.  **Load Assets:** `stuffbin` giải nén các tệp tĩnh và template vào bộ nhớ.

### Bước 2: Thiết lập Chiến dịch (Campaign Setup)
1.  Người dùng tạo mẫu (Template) bằng HTML hoặc trình kéo thả.
2.  Tạo danh sách người nhận (Lists) bằng cách import từ CSV hoặc gọi API.
3.  Soạn chiến dịch (Campaign), chọn danh sách và đặt lịch gửi (Schedule).

### Bước 3: Xử lý Gửi tin (Campaign Processing)
1.  **Manager Worker:** Luôn chạy ngầm để quét các chiến dịch ở trạng thái `scheduled` hoặc `running`.
2.  **Subscriber Fetching:** Lấy danh sách người đăng ký theo từng Batch.
3.  **Messenger Dispatch:** Đẩy email vào hàng đợi gửi qua SMTP pool hoặc HTTP Messenger.
4.  **Logging:** Ghi nhận kết quả gửi (Thành công/Thất bại).

### Bước 4: Theo dõi & Bảo trì (Tracking & Maintenance)
1.  **Tracking Events:** Khi người dùng mở email hoặc click link, các endpoint public xử lý và cập nhật vào DB.
2.  **Bounce Handling:** Quét các hộp thư Bounce (qua POP3) hoặc nhận Webhook từ SES để tự động loại bỏ các email hỏng (Blocklist).
3.  **MatView Refresh:** Chạy định kỳ (Cron) để làm mới dữ liệu biểu đồ trên Dashboard.

---

## 5. Cấu Trúc Thư Mục Chính
*   `/cmd`: Điểm khởi đầu của ứng dụng và các lệnh CLI.
*   `/internal`: Logic lõi (Auth, Bounce handling, Core CRUD, Media, Messenger).
*   `/models`: Định nghĩa các cấu trúc dữ liệu và SQL queries.
*   `/frontend`: Mã nguồn Vue.js của giao diện Admin.
*   `/static`: Các template mặc định và file tĩnh cho trang đăng ký công cộng.