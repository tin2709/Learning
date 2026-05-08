ntfy (notify) là một công cụ mã nguồn mở mạnh mẽ, cho phép gửi thông báo đẩy (push notifications) đến điện thoại hoặc máy tính thông qua các yêu cầu HTTP đơn giản (PUT/POST).

Dưới đây là phân tích chuyên sâu về công nghệ và kiến trúc của dự án ntfy:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ lập trình:** **Go (Golang)** là ngôn ngữ chủ đạo cho server và CLI. Lựa chọn này mang lại hiệu suất cao, khả năng xử lý đồng thời (concurrency) tuyệt vời thông qua Goroutines và tạo ra các file nhị phân tĩnh (static binaries) dễ dàng phân phối.
*   **Giao thức truyền tải:**
    *   **HTTP/1.1 & HTTP/2:** Sử dụng các phương thức chuẩn như `PUT`, `POST`, `GET`.
    *   **Streaming:** Hỗ trợ **EventSource (SSE)**, **WebSockets** và **NDJSON** (Newline Delimited JSON) để duy trì kết nối thời gian thực và đẩy thông báo ngay lập tức.
*   **Lưu trữ & Persistence:**
    *   **SQLite:** Cơ sở dữ liệu mặc định, nhẹ, không cần cấu hình phức tạp, phù hợp cho việc tự triển khai (self-hosting).
    *   **PostgreSQL:** Hỗ trợ cho các hệ thống quy mô lớn (như ntfy.sh), cho phép mở rộng và phân tách node xử lý/đọc dữ liệu.
    *   **S3 & Local File System:** Lưu trữ tệp đính kèm (attachments). Hệ thống sử dụng một lớp interface (`backend.go`) để trừu tượng hóa việc lưu trữ giữa Amazon S3 và đĩa cứng cục bộ.
*   **Frontend:** Được xây dựng bằng **React** kết hợp với **Material UI (MUI)**, cung cấp giao diện quản trị và nhận thông báo trực tiếp trên trình duyệt qua WebPush.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ntfy tập trung vào sự **tối giản** và **phi trạng thái (stateless-like)** cho người dùng:

*   **Mô hình Pub-Sub đơn giản:** Hệ thống hoạt động dựa trên "Topic". Bất kỳ ai biết tên Topic đều có thể gửi (Publish) hoặc nhận (Subscribe) thông báo. Không bắt buộc đăng ký tài khoản (mặc định), giúp giảm rào cản sử dụng.
*   **Trừu tượng hóa thông báo:** Một tin nhắn ntfy có thể được chuyển đổi và đẩy qua nhiều kênh khác nhau tùy cấu hình: Firebase (FCM) cho Android, APNS cho iOS, e-mail qua SMTP, hoặc thậm chí là các cuộc gọi điện thoại qua Twilio.
*   **Tối ưu cho hạ tầng tự quản (Self-hosting):** Toàn bộ server, web app và tài liệu có thể được đóng gói vào một file nhị phân duy nhất nhờ kỹ thuật `embed` của Go, giúp việc triển khai cực kỳ đơn giản (chỉ cần 1 file và 1 file config).
*   **Bảo mật đa tầng:** ntfy cung cấp hệ thống ACL (Access Control List) linh hoạt, cho phép cấu hình quyền đọc/ghi chi tiết cho từng user hoặc vô danh (anonymous) trên từng topic cụ thể.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Xử lý luồng dữ liệu (Streaming Processing):** Server sử dụng `bufio.Scanner` và các kỹ thuật streaming để đẩy dữ liệu tới khách hàng mà không làm treo tài nguyên hệ thống.
*   **Cơ chế giới hạn tốc độ (Rate Limiting):** Triển khai thuật toán **Token Bucket** (`util/limit.go`) để kiểm soát lưu lượng truy cập của khách (visitor), ngăn chặn spam và tấn công từ chối dịch vụ (DoS).
*   **Hot-reloading:** ntfy hỗ trợ tải lại cấu hình một phần (như log level) mà không cần khởi động lại server thông qua việc lắng nghe tín hiệu `SIGHUP` (`cmd/serve_unix.go`).
*   **Khả năng tương thích ngược (Gemoji):** Tích hợp hỗ trợ Emoji qua tệp JSON (`web/src/app/emojisMapped.js`), giúp biến các từ khóa như `:tada:` thành biểu tượng cảm xúc trực quan trên thông báo.
*   **Batching Queue:** Sử dụng hàng đợi gom lô (`util/batching_queue.go`) để tối ưu hóa việc ghi dữ liệu vào message cache, giảm thiểu I/O overhead khi có lượng lớn thông báo đến cùng lúc.

### 4. Luồng Hoạt động Hệ thống (System Operation Flows)

#### A. Luồng Gửi thông báo (Publish Flow):
1.  **Client** gửi request `POST /mytopic` kèm nội dung tin nhắn.
2.  **Server** kiểm tra giới hạn tốc độ (Rate Limit) và quyền truy cập (ACL).
3.  Tin nhắn được lưu vào **Cache** (SQLite/Postgres) để phục vụ các yêu cầu `since=`.
4.  **Dispatcher** quét các kết nối đang hoạt động (SSE/WebSocket) để đẩy tin nhắn đi.
5.  Nếu topic có đăng ký **FCM/WebPush**, server sẽ gọi API của Google/Apple/Browser để đẩy thông báo đến thiết bị di động.

#### B. Luồng Nhận thông báo (Subscribe Flow):
1.  **Client** mở kết nối `GET /mytopic/json` (hoặc `/sse`, `/ws`).
2.  Server giữ kết nối mở (Long-polling hoặc Streaming).
3.  Khi có tin nhắn mới, server mã hóa thành JSON và đẩy qua socket.
4.  Khách hàng nhận dữ liệu và hiển thị lên UI hoặc thực thi các script cục bộ.

#### C. Luồng Tệp đính kèm (Attachment Flow):
1.  Người dùng upload file qua `PUT`.
2.  Server kiểm tra dung lượng còn lại của người dùng (`attachment/store.go`).
3.  Dữ liệu được "pipe" trực tiếp từ HTTP request tới S3 hoặc đĩa cứng để tránh chiếm dụng RAM.
4.  Server tạo một ID duy nhất và trả về URL để người nhận có thể tải về.

### 5. Tổng kết
ntfy là một ví dụ điển hình về việc **"làm một việc và làm tốt nhất việc đó"**. Dự án kết hợp khéo léo giữa tính tiện dụng của giao thức HTTP và hiệu năng của Go để tạo ra một hệ thống thông báo thời gian thực cực kỳ linh hoạt, nhẹ nhàng nhưng vẫn đủ mạnh mẽ cho quy mô doanh nghiệp.