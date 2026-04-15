Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Stoat Backend** (một nền tảng chat mã nguồn mở, kế thừa từ Revolt), dưới đây là phân tích chi tiết:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào hiệu suất cao và khả năng mở rộng:

*   **Ngôn ngữ lập trình:** **Rust** (chiếm 93.2%), tận dụng tối đa tính an toàn bộ nhớ và hiệu năng của ngôn ngữ này.
*   **Async Runtime:** Sử dụng cả **Tokio** (phổ biến nhất trong hệ sinh thái Rust) và **async-std** (đặc biệt trong service `bonfire`).
*   **Web Framework:**
    *   **Axum:** Dùng cho các dịch vụ REST API (`delta`) và dịch vụ xử lý file.
    *   **Async-Tungstenite:** Dùng để xử lý kết nối WebSocket thời gian thực.
*   **Cơ sở dữ liệu & Lưu trữ:**
    *   **MongoDB:** Cơ sở dữ liệu chính lưu trữ dữ liệu người dùng, tin nhắn, server.
    *   **Redis:** Dùng để quản lý trạng thái hiện diện (presence), cache và làm hệ thống Pub/Sub cho các sự kiện thời gian thực.
    *   **MinIO (S3 compatible):** Lưu trữ đối tượng (file đính kèm, avatar, banner).
*   **Hệ thống tin nhắn (Messaging):** **RabbitMQ** (thông qua `amqprs`) được dùng làm Message Broker để giao tiếp giữa các dịch vụ nội bộ và xử lý tác vụ nền (như gửi Push Notification).
*   **Voice/Video:** Tích hợp **LiveKit** để xử lý truyền thông đa phương tiện.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Stoat Backend sử dụng kiến trúc **Microservices (hoặc Modular Monolith)** được tổ chức dưới dạng **Cargo Workspace**:

*   **Tách biệt Core và Service:** Các logic dùng chung (Database, Permissions, Models, Config) được đưa vào thư mục `crates/core`. Các dịch vụ chạy độc lập nằm trong `crates/services`, `crates/daemons` và các binary chính như `delta`, `bonfire`.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Đây là thành phần quan trọng nhất của một ứng dụng chat. Khi một tin nhắn được gửi qua API (`delta`), một sự kiện được xuất bản (publish) lên Redis. Server WebSocket (`bonfire`) sẽ nghe các kênh này và đẩy tin nhắn xuống các client liên quan ngay lập tức.
*   **Trừu tượng hóa cơ sở dữ liệu (Database Abstraction):** Dự án định nghĩa Trait `AbstractDatabase` (`crates/core/database/src/models/mod.rs`). Điều này cho phép hệ thống chạy với `MongoDb` thật hoặc `ReferenceDb` (giả lập trong bộ nhớ) để phục vụ kiểm thử mà không cần cài đặt database.
*   **Kiến trúc không trạng thái (Stateless):** Các dịch vụ API được thiết kế để có thể nhân bản (scale out) dễ dàng, trạng thái người dùng được quản lý tập trung qua Redis và Database.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)
*   **Trait-based Polymorphism:** Sử dụng Trait để định nghĩa hành vi cho các đối tượng database, cho phép hoán đổi driver linh hoạt.
*   **Coalescion Service:** Thư viện `revolt-coalesced` là một kỹ thuật thông minh để **gom nhóm các yêu cầu trùng lặp**. Ví dụ: nếu 1000 người cùng yêu cầu thông tin của một server trong cùng một mili giây, hệ thống chỉ thực hiện 1 truy vấn database và trả kết quả cho cả 1000 người.
*   **Custom Macros:** Sử dụng Macro để tự động hóa việc cấu hình dịch vụ (`revolt_config::configure!`) và xử lý lỗi đồng nhất.
*   **Idempotency (Tính giao hoán):** Đảm bảo các thao tác (như gửi tin nhắn) nếu bị lặp lại do lỗi mạng sẽ không tạo ra dữ liệu rác.
*   **Strong Type System:** Tận dụng hệ thống kiểu dữ liệu của Rust để định nghĩa các Permission (Quyền hạn) dưới dạng bitwise, giúp kiểm tra quyền cực nhanh.

### 4. Luồng hoạt động của hệ thống (System Workflow)

**Luồng Gửi Tin Nhắn (Message Flow):**
1.  **Client -> REST API (`delta`):** Người dùng gửi yêu cầu POST tin nhắn kèm token xác thực.
2.  **API xác thực & Lưu trữ:** `delta` kiểm tra quyền hạn, lưu tin nhắn vào MongoDB.
3.  **Pub/Sub:** `delta` xuất bản một sự kiện tin nhắn mới vào Redis và gửi một job vào RabbitMQ cho dịch vụ Push Notification.
4.  **WebSocket (`bonfire`):** Các server `bonfire` đang giữ kết nối với người nhận sẽ nhận được sự kiện từ Redis và chuyển đổi nó sang format (JSON/Msgpack) để gửi xuống client.
5.  **Push Notification (`pushd`):** Daemon này tiêu thụ job từ RabbitMQ, xác định thiết bị của người dùng và gửi thông báo qua FCM (Google) hoặc APN (Apple).

**Luồng Xử lý File (`autumn`):**
1.  **Upload:** Client upload file lên `autumn`.
2.  **Security:** `autumn` gọi ClamAV để quét virus.
3.  **Storage:** Nếu an toàn, file được mã hóa và đẩy lên MinIO/S3.
4.  **Metadata:** Thông tin file được lưu vào MongoDB để API `delta` có thể tham chiếu.

### Tổng kết
Stoat Backend là một hệ thống chat được thiết kế rất bài bản. Việc chọn **Rust** làm ngôn ngữ chủ đạo kết hợp với **Redis/RabbitMQ** cho thấy nhà phát triển ưu tiên sự ổn định, tốc độ xử lý thời gian thực và khả năng chịu tải lớn. Kiến trúc phân tách rõ ràng giữa API (REST) và Real-time (WebSocket) giúp hệ thống linh hoạt trong việc bảo trì và nâng cấp.