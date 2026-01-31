Dựa trên nội dung mã nguồn của dự án **FileRise**, dưới đây là bản phân tích chi tiết theo cấu trúc chuyên nghiệp, tập trung vào các đặc tính cốt lõi của hệ thống quản lý tệp tin này:

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **PHP 8.3+**. Sử dụng các tính năng hiện đại như `strict_types` và các thư viện mã hóa tiên tiến.
*   **Kiến trúc dữ liệu (Zero-Database):** Điểm đặc trưng nhất là ứng dụng **không cần cơ sở dữ liệu SQL** (MySQL/PostgreSQL). Toàn bộ thông tin người dùng, phân quyền (ACL) và metadata được lưu trữ dưới dạng tệp **JSON** và **Text** (`users.txt`, `adminConfig.json`).
*   **Giao thức truyền tải:**
    *   **REST API:** Hệ thống cung cấp API đầy đủ với tài liệu OpenAPI/Swagger (ReDoc).
    *   **WebDAV:** Sử dụng thư viện **sabre/dav** để cho phép người dùng gắn FileRise như một ổ đĩa mạng trên máy tính cá nhân.
*   **Bảo mật & Mã hóa:**
    *   **libsodium:** Cung cấp cơ chế mã hóa dữ liệu tại chỗ (Encryption at rest) chuẩn quân đội.
    *   **OIDC (OpenID Connect):** Hỗ trợ SSO qua các nền tảng như Authentik, Keycloak hoặc Auth0.
    *   **TOTP:** Xác thực hai yếu tố (2FA) cho người dùng nội bộ.
*   **Giao diện (Frontend):**
    *   **Vanilla JS & Bootstrap 4.6:** Đảm bảo tốc độ tải trang cực nhanh mà không cần các framework nặng.
    *   **Resumable.js:** Xử lý tải lên các tệp tin cực lớn thông qua kỹ thuật chia nhỏ phân đoạn (chunking).
    *   **CodeMirror:** Trình soạn thảo mã nguồn tích hợp ngay trên trình duyệt.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án được thiết kế với triết lý **"File-First"** (Ưu tiên tệp tin) và tính linh hoạt tối đa:

*   **Tách biệt logic lưu trữ (Storage Abstraction):** Sử dụng mẫu thiết kế **Adapter**. Mọi thao tác tệp đều thông qua một Interface chung, cho phép hệ thống kết nối với nhiều nguồn khác nhau (Local, S3, SMB, SFTP, Google Drive) mà không thay đổi logic xử lý chính.
*   **Nhất quán trên mọi giao diện:** Một tư duy quan trọng là hệ thống phân quyền (ACL) được thực thi đồng bộ 100% trên cả giao diện Web, API và WebDAV. Điều này đảm bảo an toàn dữ liệu dù người dùng truy cập bằng cách nào.
*   **Kiến trúc mở rộng (Hook-based Extension):** Phiên bản Pro được nhúng vào Core thông qua các điểm nối (hooks) và bootstrap. Điều này giúp duy trì mã nguồn Core sạch sẽ trong khi vẫn cho phép các tính năng nâng cao hoạt động mượt mà.
*   **Tối ưu hóa khả năng mở rộng (Scalability):** Thay vì quét toàn bộ ổ cứng theo thời gian thực (gây chậm trễ), FileRise sử dụng cơ chế **Metadata Indexing**. Hệ thống quét và lưu cấu trúc thư mục vào bộ nhớ đệm JSON, cho phép xử lý các cây thư mục lên tới 100.000+ mục mà vẫn phản hồi tức thì.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Tải lên phân đoạn & Tiếp tục (Chunked & Resumable Uploads):** Tệp lớn được chia thành các mảnh nhỏ. Nếu mất mạng, hệ thống chỉ cần tải lên những mảnh còn thiếu thay vì bắt đầu lại từ đầu.
*   **Mã hóa dữ liệu tại chỗ (Transparent Encryption):** Sử dụng thuật toán `secretstream` của libsodium. Tệp được mã hóa ngay khi ghi xuống đĩa và chỉ được giải mã khi người dùng có quyền hợp lệ yêu cầu tải về.
*   **Kế thừa quyền hạn tinh vi (Granular ACL & Inheritance):** Quyền hạn (Xem, Tải lên, Xóa, Chia sẻ...) được thiết lập theo từng thư mục và tự động kế thừa xuống các thư mục con, cho phép quản lý hàng ngàn người dùng với các mức độ truy cập khác nhau.
*   **Nhận diện Proxy ngược (Reverse Proxy Awareness):** Kỹ thuật xử lý các Header `X-Forwarded-*` để hoạt động hoàn hảo sau Nginx, Traefik hoặc Cloudflare, đảm bảo các liên kết chia sẻ luôn đúng giao thức (HTTPS) và IP của khách hàng được ghi lại chính xác.
*   **Quét Virus tích hợp:** Tích hợp trực tiếp với **ClamAV** để tự động quét các tệp tải lên, bảo vệ hệ thống khỏi mã độc.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Xác thực (Authentication):**
    *   Người dùng đăng nhập qua Form, SSO (OIDC) hoặc Header từ Proxy.
    *   Hệ thống cấp Session và Token CSRF để bảo vệ mọi yêu cầu sau đó.
2.  **Kiểm soát truy cập (ACL Check):**
    *   Mỗi khi có yêu cầu (ví dụ: Xem tệp), hệ thống sẽ nạp cấu hình quyền từ `userPermissions.json` (đã mã hóa).
    *   Kiểm tra quyền của người dùng tại thư mục mục tiêu. Nếu hợp lệ, yêu cầu mới được chuyển đến lớp lưu trữ.
3.  **Xử lý lưu trữ (Storage Operation):**
    *   `StorageFactory` xác định loại Adapter cần dùng (nội bộ hoặc đám mây).
    *   Nếu là thư mục mã hóa, luồng dữ liệu sẽ đi qua bộ giải mã trước khi trả về cho người dùng.
4.  **Cập nhật Metadata & Ghi nhật ký (Metadata & Audit):**
    *   Sau mỗi thao tác thành công, hệ thống cập nhật lại các tệp chỉ mục JSON trong thư mục `metadata/`.
    *   (Bản Pro) Ghi lại chi tiết hành động vào nhật ký hệ thống (Audit Logs) để phục vụ quản trị.

### Tổng kết
**FileRise** là một minh chứng cho việc ứng dụng PHP hiện đại có thể mạnh mẽ, an toàn và nhanh như thế nào mà không cần đến sự phức tạp của cơ sở dữ liệu truyền thống. Đây là kiến trúc hoàn hảo cho các giải pháp lưu trữ riêng tư, Homelab và cổng thông tin khách hàng (Client Portals).