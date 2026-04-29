Dựa trên cấu trúc mã nguồn và các tài liệu liên quan đến dự án **Sync-in Server**, dưới đây là phân tích chi tiết về kỹ thuật và kiến trúc của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)

Hệ thống được xây dựng theo mô hình Monorepo sử dụng npm workspaces, chia thành hai phần chính:

*   **Backend (NestJS + Fastify):**
    *   **Framework:** NestJS (TypeScript) kết hợp với **Fastify** làm HTTP engine thay vì Express để tối ưu hiệu năng và thông lượng (throughput).
    *   **ORM & Database:** **Drizzle ORM** kết hợp với MySQL/MariaDB. Đây là một ORM hiện đại, kiểu chữ (type-safe) giúp truy vấn SQL nhanh và minh bạch.
    *   **Real-time:** **Socket.io** (WebSockets) để xử lý thông báo, bình luận và cập nhật trạng thái file tức thời.
    *   **Search & Indexing:** Tích hợp bộ máy lập chỉ mục (Indexing) cho tài liệu, hỗ trợ **OCR** (nhận dạng ký tự quang học) thông qua **Tesseract.js** để tìm kiếm nội dung bên trong file PDF và hình ảnh.
*   **Frontend (Angular):**
    *   Sử dụng Angular với kiến trúc Component-based hiện đại, SCSS cho giao diện và RxJS để xử lý các luồng dữ liệu bất đồng bộ.
*   **Hợp tác & Soạn thảo:**
    *   Tích hợp sâu với **Collabora Online** và **OnlyOffice** thông qua giao thức WOPI hoặc các Adapter tùy chỉnh để hỗ trợ đồng soạn thảo văn bản trực tuyến.
*   **Xác thực:**
    *   Hỗ trợ đa dạng: Local, **LDAP/AD**, và **OpenID Connect (OIDC)** cho SSO doanh nghiệp. Tích hợp MFA (Multi-Factor Authentication).

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc hướng Module (Modular Architecture):**
    *   Mỗi tính năng là một module độc lập trong NestJS (`admin`, `files`, `users`, `spaces`, `shares`, `sync`). Điều này cho phép mở rộng hoặc thay thế logic từng phần mà không ảnh hưởng toàn cục.
*   **Phân tầng dữ liệu (Separation of Concerns):**
    *   Dự án phân tách rõ ràng giữa **Services** (logic nghiệp vụ), **Queries** (truy vấn dữ liệu chuyên sâu) và **Managers** (điều phối hoạt động giữa các module).
*   **Mô hình Quản lý Không gian (Spaces & Shares):**
    *   Kiến trúc không chỉ là cây thư mục đơn giản mà dựa trên các "Space" (không gian làm việc) và "Share" (chia sẻ) với hệ thống phân quyền chi tiết (Fine-grained permissions) thông qua `SpaceGuard`.
*   **Thiết kế Ưu tiên Tự lưu trữ (Self-hosted by Design):**
    *   Tất cả cấu hình được xử lý qua biến môi trường (environment variables) và Docker, giúp triển khai dễ dàng trên hạ tầng riêng (on-premise).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Kỹ thuật Khóa file (File Locking):** Một cơ chế quan trọng để ngăn chặn xung đột khi nhiều người cùng sửa một file hoặc khi file đang được đồng bộ hóa với Desktop Client.
*   **Lập chỉ mục tăng trưởng (Incremental Indexing):** Hệ thống lắng nghe các `FileEvent` để cập nhật chỉ mục tìm kiếm ngay khi file bị thay đổi, giúp kết quả tìm kiếm luôn mới nhất mà không cần quét lại toàn bộ ổ cứng.
*   **Doc-Textify:** Một lớp chuyển đổi (Adapter) cho phép trích xuất văn bản từ nhiều định dạng khác nhau (Word, Excel, Markdown, PDF) để phục vụ cho tính năng Full-text search.
*   **Tối ưu hóa Đồng bộ (Sync Optimization):** Sử dụng Gzip để nén các gói dữ liệu sai biệt (diff) khi gửi về cho Desktop Client, giúp tiết kiệm băng thông.
*   **Quản lý Quota (Storage Quota):** Cơ chế tính toán dung lượng thực tế của người dùng và các không gian chung theo thời gian thực để ngăn chặn việc sử dụng quá tải bộ nhớ.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Xác thực:** Người dùng đăng nhập qua Web, Desktop hoặc CLI. Hệ thống cấp phát JWT (Access/Refresh Tokens).
2.  **Truy cập tài nguyên:**
    *   Khi người dùng mở một Space, `SpaceGuard` sẽ kiểm tra quyền dựa trên DB.
    *   Nếu truy cập qua WebDAV, hệ thống sử dụng một lớp `WebDAVModule` để chuyển đổi các phương thức HTTP đặc thù (PROPFIND, MKCOL, LOCK) thành các lệnh xử lý file tương ứng.
3.  **Xử lý File:**
    *   Khi upload: File được lưu vào `dataPath`, đồng thời ghi Metadata vào DB. Một sự kiện được kích hoạt để `FilesContentIndexer` bắt đầu trích xuất văn bản/OCR dưới nền.
4.  **Cộng tác:**
    *   Khi mở tài liệu trực tuyến: Backend thiết lập phiên làm việc với OnlyOffice/Collabora. File được "khóa" tạm thời. Mọi thay đổi được lưu lại và kích hoạt thông báo real-time qua WebSocket tới những người dùng liên quan.
5.  **Đồng bộ hóa:**
    *   Desktop Client gửi yêu cầu kiểm tra thay đổi -> Server tính toán sự khác biệt giữa trạng thái Local và Server -> Trả về danh sách hành động (tải về, xóa, ghi đè).

**Kết luận:** **Sync-in** là một nền tảng thay thế mạnh mẽ cho Nextcloud/Dropbox với tư duy công nghệ hiện đại, tập trung vào tốc độ xử lý (Fastify), tính ổn định của dữ liệu (Drizzle SQL) và khả năng tìm kiếm nội dung sâu (OCR/Indexing).