Dựa trên mã nguồn và tài liệu của dự án **MicroBin**, dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

MicroBin được xây dựng với mục tiêu cực kỳ nhẹ, hiệu suất cao và "tất cả trong một" (self-contained).

*   **Ngôn ngữ lập trình:** **Rust** - Đảm bảo an toàn bộ nhớ, tốc độ thực thi tương đương C++ nhưng không cần runtime nặng nề.
*   **Web Framework:** **Actix-web** - Một trong những framework web nhanh nhất hiện nay, hoạt động dựa trên mô hình Actor và xử lý bất đồng bộ (async).
*   **Templating Engine:** **Askama** - Sử dụng template được biên dịch trực tiếp vào mã máy của Rust. Điều này giúp MicroBin không tốn tài nguyên để "parse" HTML lúc runtime và phát hiện lỗi template ngay khi biên dịch.
*   **Cơ sở dữ liệu:** 
    *   **SQLite (mặc định):** Sử dụng thông qua `rusqlite`.
    *   **JSON:** Tùy chọn lưu trữ file đơn giản cho các hệ thống cực nhỏ.
    *   Lớp trừu tượng (`util/db.rs`) cho phép chuyển đổi giữa hai loại này mà không ảnh hưởng logic nghiệp vụ.
*   **Đóng gói dữ liệu:** **Rust-embed** - Toàn bộ tài liệu tĩnh (CSS, JS, Logo) được nhúng thẳng vào file thực thi duy nhất.
*   **Mã hóa:**
    *   Server-side: `magic-crypt` (AES-256).
    *   Client-side: `aes-js` (JavaScript) thực hiện mã hóa E2E ngay tại trình duyệt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của MicroBin xoay quanh triết lý **"Micro-service in a box"**:

*   **Tính tự đóng gói (Self-containment):** MicroBin không cần máy chủ database riêng (như MySQL/Postgres), không cần máy chủ web riêng (như Nginx). Mọi thứ cần thiết để chạy một dịch vụ chia sẻ file đều nằm trong một file binary vài MB.
*   **Mô hình Dữ liệu (The Pasta Model):** Mọi thực thể (text, file, URL) được gọi là một "Pasta". Điều này giúp đồng nhất hóa cách quản lý, thiết lập thời gian hết hạn (expiry) và cơ chế tự hủy (burn-after-read).
*   **Trình định danh thân thiện:** Thay vì sử dụng UUID dài loằng ngoằng, MicroBin sử dụng `Animal Names` (tên động vật kết hợp - ví dụ: `pig-dog-cat`) hoặc `Hashids` để tạo ra các URL ngắn, dễ đọc và dễ nhớ cho người dùng.
*   **Tách biệt logic hiển thị và API:** Các endpoint được module hóa rõ rệt trong thư mục `src/endpoints/`, tách biệt giữa việc render HTML cho người dùng và phục vụ dữ liệu thô (raw) cho máy móc/script.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Quản lý trạng thái chia sẻ (Shared State):** Sử dụng `actix_web::web::Data` bọc quanh một `AppState` chứa `Mutex<Vec<Pasta>>`. Điều này cho phép truy cập danh sách metadata cực nhanh từ bộ nhớ RAM nhưng vẫn đảm bảo an toàn đa luồng (thread-safety).
*   **Xử lý Multipart Streaming:** Khi tải file lên, MicroBin không tải toàn bộ file vào RAM mà "stream" dữ liệu từ multipart request trực tiếp xuống ổ đĩa. Kỹ thuật này giúp hệ thống xử lý được file lớn ngay cả trên máy chủ chỉ có 128MB RAM.
*   **Hybrid Encryption (Mã hóa hỗn hợp):**
    *   *Private Mode:* Server nhận mật khẩu, mã hóa nội dung bằng AES-256 rồi mới lưu.
    *   *Secret Mode (E2E):* Browser chạy JavaScript mã hóa nội dung bằng mật khẩu của người dùng. Server chỉ nhận và lưu trữ "ciphertext" (rác dữ liệu). Server hoàn toàn không biết nội dung gốc hay mật khẩu.
*   **Syntax Highlighting linh hoạt:** Hỗ trợ cả `Syntect` (server-side highlight) để phục vụ các trình duyệt không chạy JS, và `Highlight.js` (client-side) để giảm tải cho CPU của server.

### 4. Luồng hoạt động hệ thống (System Workflows)

#### Luồng tải lên (Upload Flow):
1.  Người dùng gửi request Multipart (Text/File/Settings).
2.  `create::create` xử lý:
    *   Kiểm tra mật khẩu uploader (nếu có).
    *   Mã hóa (nếu chọn Private/Secret).
    *   Tính toán thời gian hết hạn (Timestamp).
    *   Lưu metadata vào database và lưu file vào thư mục `attachments`.
3.  Cấp một `owner_token` qua Cookie (có hiệu lực 15 giây) để người tạo có thể xem lại ngay mà không bị tính là một lượt đọc (read count).

#### Luồng truy xuất và Tự hủy (Viewing & Burn Flow):
1.  Người dùng truy cập URL (ví dụ: `/upload/ape-lion`).
2.  Hệ thống chuyển đổi tên động vật thành ID số.
3.  Tìm trong DB:
    *   Nếu hết hạn: Xóa ngay lập tức và trả về 404.
    *   Nếu có cơ chế tự hủy (`burn_after_reads`): Kiểm tra `read_count`. Nếu lượt đọc hiện tại vượt ngưỡng, hệ thống sẽ thực hiện xóa metadata và file vật lý khỏi ổ đĩa ngay trong quá trình xử lý request đó.
4.  Render template thông qua Askama và trả về trình duyệt.

#### Luồng dọn rác (GC - Garbage Collection):
MicroBin không có tiến trình chạy nền (background daemon) phức tạp. Thay vào đó, mỗi khi có request truy cập vào danh sách (`/list`) hoặc quản trị, hàm `remove_expired` sẽ được gọi để quét và dọn dẹp các Pasta đã hết hạn, đảm bảo ổ đĩa không bị đầy bởi dữ liệu cũ.

### Tổng kết
MicroBin là minh chứng cho việc **Rust** có thể tạo ra các công cụ mạng mạnh mẽ với kích thước siêu nhỏ. Sự kết hợp giữa tính an toàn của Rust và tốc độ của Actix-web giúp nó trở thành một giải pháp chia sẻ dữ liệu nội bộ lý tưởng, cân bằng giữa tiện nghi và bảo mật tuyệt đối.