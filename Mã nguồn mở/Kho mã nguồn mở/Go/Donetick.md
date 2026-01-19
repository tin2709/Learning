Chào bạn, dưới đây là bản phân tích chi tiết về dự án **Donetick** dựa trên mã nguồn bạn đã cung cấp, trình bày bằng tiếng Việt:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án Donetick được xây dựng trên một ngăn xếp công nghệ hiện đại, chú trọng vào hiệu suất và khả năng tự vận hành (self-hosted):

*   **Ngôn ngữ lập trình:** **Go (Golang)** phiên bản 1.24. Đây là lựa chọn tối ưu cho các ứng dụng backend cần tốc độ thực thi cao và tiêu tốn ít tài nguyên.
*   **Web Framework:** **Gin Gonic**. Một framework web cực kỳ phổ biến trong hệ sinh thái Go, giúp xây dựng RESTful API nhanh chóng với hiệu năng cao.
*   **Dependency Injection (DI):** **Uber fx**. Đây là điểm nhấn quan trọng giúp quản lý vòng đời ứng dụng và các phụ thuộc giữa các thành phần (database, repository, service) một cách linh hoạt và dễ kiểm thử.
*   **Database & ORM:** Sử dụng **GORM** làm bộ ánh xạ quan hệ đối tượng, hỗ trợ nhiều loại database như **SQLite** (mặc định cho self-hosted) và **PostgreSQL/MySQL**.
*   **Authentication & Security:** 
    *   **JWT (JSON Web Token):** Quản lý phiên đăng nhập.
    *   **MFA (Multi-Factor Authentication):** Hỗ trợ TOTP và mã dự phòng.
    *   **OAuth2/OIDC:** Tích hợp đăng nhập bên thứ ba (Google, Apple, Authentik...).
*   **Real-time:** Hỗ trợ cả **WebSockets** và **SSE (Server-Sent Events)** để đồng bộ dữ liệu tức thời giữa các thiết bị.
*   **Cloud Storage:** Tương thích với các dịch vụ S3 (AWS S3, Cloudflare R2, MinIO) để lưu trữ hình ảnh.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Donetick đi theo hướng **Modular Monolith** kết hợp với các nguyên tắc của **Clean Architecture**:

*   **Tách biệt trách nhiệm (Separation of Concerns):** Mã nguồn được chia thành các package rõ ràng:
    *   `internal/auth`: Xử lý xác thực.
    *   `internal/chore`: Logic chính về quản lý công việc và lịch trình.
    *   `internal/circle`: Quản lý nhóm người dùng (Circles).
    *   `internal/repo`: Tầng giao tiếp với database (Repository Pattern), giúp tách biệt logic nghiệp vụ khỏi các truy vấn SQL.
*   **Ưu tiên khả năng mở rộng:** Nhờ sử dụng Uber fx, việc thêm các module mới (như các loại thông báo mới hoặc tích hợp bên thứ ba) trở nên rất dễ dàng mà không làm ảnh hưởng đến cấu trúc hiện có.
*   **Thiết kế cho Self-hosted:** Mọi cấu hình từ database, port, logging đến các dịch vụ bên thứ ba đều được quản lý qua file YAML (`selfhosted.yaml`) hoặc biến môi trường (Environment Variables), giúp người dùng dễ dàng triển khai qua Docker.

---

### 3. Các kỹ thuật then chốt (Key Techniques)

Donetick áp dụng nhiều kỹ thuật lập trình nâng cao để giải quyết các bài toán thực tế:

*   **Lập lịch thông minh (Advanced Scheduling):** 
    *   Hệ thống xử lý được các mẫu lặp lại phức tạp: theo ngày, tuần, tháng, quý, hoặc theo các thứ cụ thể trong tuần (ví dụ: thứ Hai đầu tiên của tháng).
    *   **Adaptive Scheduling:** Sử dụng thuật toán tính toán độ trễ trung bình từ lịch sử hoàn thành để gợi ý ngày đến hạn tiếp theo.
*   **Xử lý ngôn ngữ tự nhiên (NLP):** (Đề cập trong README) Trích xuất thông tin ngày giờ từ văn bản thô khi tạo tác vụ.
*   **Chiến lược gán việc (Assignee Rotation):** Hỗ trợ nhiều chế độ như gán ngẫu nhiên, gán cho người ít việc nhất, hoặc xoay vòng (Round-robin).
*   **Impersonation (Mạo danh):** Cho phép Admin/Manager thao tác dưới danh nghĩa người dùng khác trong cùng nhóm để hỗ trợ hoặc quản lý (thường thấy trong các hệ thống doanh nghiệp).
*   **Middleware Patterns:** Sử dụng hệ thống middleware của Gin để xử lý các vấn đề cắt ngang (cross-cutting concerns) như: giới hạn lưu lượng (Rate Limiting), kiểm tra quyền Plus Member, logging và xử lý timeout.
*   **Security Guard:** Một kỹ thuật thú vị là hệ thống tự động kiểm tra độ mạnh của JWT Secret khi khởi động và sẽ "panic" (dừng ứng dụng) nếu phát hiện secret quá yếu để bảo vệ người dùng.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow Summary)

Luồng hoạt động của dự án có thể tóm tắt qua các bước sau:

1.  **Khởi tạo (Bootstrap):** 
    *   `main.go` khởi chạy, sử dụng `Viper` để đọc cấu hình từ file YAML hoặc biến môi trường. 
    *   `Uber fx` được kích hoạt để khởi tạo Database, thực hiện Migration (cập nhật cấu trúc bảng) và đăng ký các Handler cho API.
2.  **Xác thực (Authentication):**
    *   Người dùng đăng nhập qua API. Hệ thống kiểm tra mật khẩu (đã hash bằng `bcrypt`) hoặc qua OAuth2.
    *   Nếu có MFA, một session tạm thời được tạo ra yêu cầu mã OTP trước khi cấp JWT chính thức.
3.  **Xử lý tác vụ (Task/Chore Processing):**
    *   Khi người dùng tạo/sửa tác vụ, dữ liệu được lưu vào database. 
    *   Nếu tác vụ lặp lại, tầng logic `chore/scheduler.go` sẽ tính toán ngày đến hạn tiếp theo (`NextDueDate`) dựa trên các metadata về tần suất.
4.  **Hệ thống thông báo (Notification & Events):**
    *   Một bộ lập lịch (Scheduler) chạy ngầm kiểm tra các tác vụ sắp đến hạn.
    *   Các sự kiện (events) được bắn ra qua Webhook hoặc gửi thông báo trực tiếp tới Telegram/Discord/Pushover.
5.  **Đồng bộ thời gian thực (Real-time Sync):**
    *   Bất kỳ thay đổi nào (thêm, sửa, hoàn thành tác vụ) sẽ được `RealTimeService` đẩy tới tất cả các client đang kết nối thông qua WebSocket hoặc SSE để cập nhật giao diện ngay lập tức mà không cần load lại trang.
6.  **Triển khai (Deployment):**
    *   Ứng dụng được đóng gói trong Docker. File `Dockerfile` sử dụng kỹ thuật multi-stage build để tối ưu hóa dung lượng image cuối cùng (chỉ chứa binary và cấu hình).

Tổng quan, **Donetick** là một dự án Go rất bài bản, áp dụng các best practices về kiến trúc phần mềm, phù hợp làm mẫu để học tập về cách xây dựng ứng dụng backend chuyên nghiệp.