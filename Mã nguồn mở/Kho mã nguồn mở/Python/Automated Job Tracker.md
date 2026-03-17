Chào bạn, đây là bản phân tích chi tiết về kiến trúc và công nghệ của dự án **Automated Job Tracker** dựa trên mã nguồn bạn đã cung cấp:

### 1. Công nghệ cốt lõi (Core Technologies)

Hệ thống được xây dựng trên một ngăn xếp (stack) Python hiện đại, tập trung vào việc tích hợp các dịch vụ bên thứ ba:

*   **Framework chính:** **Django 5.1.6** phối hợp với **Django REST Framework (DRF)**. Đây là lựa chọn mạnh mẽ để xây dựng API ổn định, bảo mật và có khả năng mở rộng.
*   **Xử lý tác vụ bất đồng bộ:** **Celery 5.5.2** và **RabbitMQ** (Message Broker). Đây là thành phần quan trọng nhất để xử lý việc quét email và gọi API AI mà không làm treo giao diện người dùng.
*   **Trí tuệ nhân tạo (AI):** **OpenAI API (GPT-4o-mini)**. Sử dụng mô hình ngôn ngữ lớn để thay thế các tập lệnh Regex phức tạp, giúp bóc tách thông tin (tên công ty, vị trí, trạng thái) từ các email có định dạng không cố định.
*   **Tích hợp Google Workspace:** **Gmail API** và **Google Sheets API**. Hệ thống sử dụng thư viện `google-api-python-client` để tương tác trực tiếp với hộp thư và bảng tính của người dùng.
*   **Cơ sở dữ liệu:** **PostgreSQL** cho môi trường Production (đảm bảo tính toàn vẹn dữ liệu) và SQLite cho môi trường Mock/Test.
*   **Xác thực:** **JWT (JSON Web Token)** kết hợp với **Google OAuth 2.0**. Đặc biệt, dự án sử dụng kỹ thuật lưu JWT trong **HTTP-only Cookies** để tăng cường bảo mật chống lại tấn công XSS.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Dự án đi theo mô hình **Distributed Task Queue Architecture** (Kiến trúc hàng đợi tác vụ phân tán):

*   **Tách biệt logic nghiệp vụ (Service Layer):** Thay vì viết logic trong `views.py`, tác giả tách nhỏ ra thành `email_services.py` và `googlesheet_services.py`. Tư duy này giúp code dễ kiểm thử (Unit Test) và tái sử dụng.
*   **Thiết kế bất đồng bộ (Non-blocking design):** Việc quét email có thể mất vài phút. Thay vì bắt client chờ phản hồi HTTP, backend tạo ra một "Job" (Celery Task) và trả về ngay lập tức một `task_id`. Người dùng có thể tiếp tục làm việc khác trong khi Task chạy ngầm.
*   **Tư duy "Stateless" cho API:** Backend xác thực qua Token, không duy trì Session trên Server, giúp hệ thống nhẹ hơn và dễ triển khai trên các nền tảng như Heroku hay Docker.
*   **Tích hợp AI theo hướng "Structured Output":** Dự án sử dụng kỹ thuật ép kiểu phản hồi từ AI (JSON Schema) trong file `parsers.py`, đảm bảo dữ liệu trả về từ GPT luôn ở định dạng JSON chuẩn để code có thể xử lý tiếp mà không bị lỗi.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Quản lý OAuth Token:** Hệ thống lưu trữ cả `access_token` và `refresh_token` của Google vào database. Khi `access_token` hết hạn, backend tự động dùng `refresh_token` để lấy token mới, đảm bảo trải nghiệm người dùng không bị gián đoạn.
*   **Xử lý nội dung Email (HTML Parsing):** Sử dụng **BeautifulSoup4** để làm sạch nội dung email, chuyển đổi từ HTML sang Plain Text trước khi gửi đến OpenAI nhằm tiết kiệm Token (chi phí API) và tăng độ chính xác cho AI.
*   **Batch Processing (Xử lý theo lô):** Trong `email_services.py`, email được xử lý theo từng lô (mặc định 10 email) thay vì xử lý toàn bộ một lúc, giúp kiểm soát tốt bộ nhớ và tránh bị giới hạn (rate limit) bởi các API bên ngoài.
*   **Đồng bộ hóa đa nền tảng:** Kỹ thuật ghi dữ liệu song song: ghi vào Database cục bộ để quản lý và đồng thời ghi vào Google Sheets qua API để người dùng xem trực tiếp.
*   **Bảo mật JWT qua Cookie:** Việc tùy chỉnh `CookieJWTAuthentication` để đọc token từ cookie thay vì Header giúp bảo vệ ứng dụng web tốt hơn, vì JavaScript ở frontend không thể truy cập vào HTTP-only cookie này.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Giai đoạn Xác thực:** Người dùng đăng nhập qua Google -> Backend nhận `code` -> Trao đổi lấy Tokens -> Tạo User và phát hành JWT (lưu vào Cookie) -> Chuyển hướng người dùng về Dashboard.
2.  **Giai đoạn Kết nối:** Người dùng dán URL Google Sheet -> Backend dùng Regex bóc tách `spreadsheet_id` và lưu vào hồ sơ người dùng.
3.  **Giai đoạn Kích hoạt:** Người dùng ấn "Fetch Job" -> Một request POST gửi đến backend -> Backend đẩy một Task vào hàng đợi RabbitMQ -> Trả về `task_id` cho Frontend.
4.  **Giai đoạn Xử lý ngầm (Worker):**
    *   Celery Worker lấy email từ Gmail API sau ngày fetch gần nhất.
    *    BeautifulSoup trích xuất text -> OpenAI phân loại xem có phải email tìm việc không.
    *   Nếu đúng, trích xuất thông tin -> Lưu vào PostgreSQL.
    *   Cập nhật thông tin đó vào hàng tiếp theo trên Google Sheet.
5.  **Giai đoạn Hoàn tất:** Frontend liên tục gọi (polling) API `task_status` -> Khi thấy trạng thái `SUCCESS`, hiển thị thông báo thành công và cập nhật danh sách công việc trên màn hình.

Dự án này là một ví dụ điển hình về việc kết hợp sức mạnh của **Web Framework truyền thống (Django)**, **Xử lý tác vụ nền (Celery)** và **Trí tuệ nhân tạo (OpenAI)** để giải quyết một bài toán thực tế.