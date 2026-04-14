Dựa trên các tệp tin mã nguồn của dự án **Slink** (Image Sharing Platform), dưới đây là bản phân tích chi tiết về Công nghệ cốt lõi, Kiến trúc, Kỹ thuật lập trình và Luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một "stack" công nghệ rất hiện đại và tối ưu cho hiệu suất cao:

*   **Backend:**
    *   **PHP 8.5+:** Sử dụng các tính năng mới nhất của PHP (như `property hooks` trong mã nguồn).
    *   **Symfony 8.0:** Framework chính để xây dựng API và quản lý hệ thống.
    *   **FrankenPHP:** Một ứng dụng máy chủ PHP hiện đại (viết bằng Go), tích hợp sẵn Caddy, hỗ trợ chế độ `worker mode` giúp tăng tốc độ phản hồi API cực nhanh.
    *   **EventSauce:** Thư viện chuyên biệt cho **Event Sourcing** trong PHP.
    *   **libvips (via php-vips):** Thư viện xử lý hình ảnh cực nhanh, tiết kiệm RAM hơn nhiều so với GD hay Imagick, dùng để resize và nén ảnh.
    *   **Mercure:** Giao thức dùng để đẩy dữ liệu thời gian thực (Real-time) từ server xuống trình duyệt qua SSE (Server-Sent Events).
*   **Frontend:**
    *   **SvelteKit & Svelte 5:** Framework frontend hiện đại với cơ chế Reactivity mới (Runes).
    *   **TypeScript:** Đảm bảo an toàn kiểu dữ liệu cho toàn bộ logic frontend.
    *   **Vite:** Công cụ đóng gói mã nguồn nhanh.
*   **Infrastructure:**
    *   **Docker & Docker Bake:** Quản lý container hóa và build ảnh đa kiến trúc (amd64/arm64).
    *   **Redis:** Dùng làm Cache, Message Broker cho Symfony Messenger và quản lý Session.
    *   **Storage Providers:** Hỗ trợ linh hoạt Local, SMB (Samba), và AWS S3.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Slink cực kỳ bài bản, áp dụng các mô hình thiết kế phần mềm cao cấp:

*   **Domain-Driven Design (DDD):** Mã nguồn được tổ chức theo các "Domain" (Bookmark, Collection, Image, User, Tag...). Mỗi Domain chứa logic nghiệp vụ riêng, tách biệt hoàn toàn với hạ tầng.
*   **Event Sourcing (Tiếp cận theo dòng sự kiện):** Thay vì chỉ lưu trạng thái hiện tại, Slink lưu lại toàn bộ lịch sử thay đổi thông qua các "Events". Ví dụ: Thay vì chỉ có `ImageView`, hệ thống lưu `ImageWasCreated`, `ImageWasTagged`. Điều này cho phép phục hồi dữ liệu và kiểm vết (audit trail) hoàn hảo.
*   **CQRS (Command Query Responsibility Segregation):**
    *   **Commands:** Xử lý việc ghi dữ liệu (ví dụ: `UploadImageCommand`, `CreateCommentCommand`).
    *   **Queries:** Xử lý việc đọc dữ liệu (ví dụ: `GetImageListQuery`).
    *   Kiến trúc này giúp tối ưu hóa hiệu suất đọc/ghi riêng biệt.
*   **Hexagonal Architecture (Kiến trúc lục lăng):** Tách biệt rõ ràng giữa lớp **Application** (điều phối), **Domain** (nghiệp vụ lõi) và **Infrastructure** (kết nối cơ sở dữ liệu, file system, thư viện bên thứ ba).

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Resource Processors & Data Providers:** Hệ thống sử dụng một mô hình mở rộng để xử lý dữ liệu API. Ví dụ, `ImageResourceProcessor` sẽ tự động gọi các `DataProvider` (như `BookmarkDataProvider`, `CollectionDataProvider`) để đính kèm thêm thông tin vào ảnh (như ảnh này đã được bookmark chưa, thuộc collection nào) mà không làm rối mã nguồn chính.
*   **Rule-based Sanitization:** Xử lý bảo mật file SVG thông qua một loạt các quy tắc (`SanitizationRuleInterface`), giúp loại bỏ mã độc, script hoặc các tham chiếu bên ngoài trong file ảnh.
*   **Deduplication (Khử trùng dữ liệu):** Khi upload, hệ thống tính toán mã hash của file. Nếu file đã tồn tại trên server, nó sẽ ánh xạ ảnh mới vào file cũ để tiết kiệm không gian lưu trữ.
*   **HMAC URL Signing:** Sử dụng chữ ký số để bảo vệ các URL hình ảnh, đảm bảo các tham số như size, format không bị giả mạo bởi người dùng.
*   **Process Supervisor (s6-overlay):** Trong Docker, dự án sử dụng s6-overlay để quản lý đồng thời nhiều tiến trình (PHP, Redis, Caddy) trong một container duy nhất, đảm bảo tính ổn định và tự khởi động lại khi lỗi.

### 4. Luồng hoạt động của hệ thống (System Workflow)

#### Luồng Upload và Xử lý ảnh:
1.  **Client:** Gửi yêu cầu POST kèm file ảnh và các tùy chọn (tags, visibility).
2.  **API (Symfony):** `UploadImageHandler` tiếp nhận.
3.  **Validation:** Kiểm tra dung lượng, định dạng và tính hợp lệ của file.
4.  **Deduplication:** Tính hash. Nếu trùng, sử dụng lại file cũ. Nếu mới, `VipsImageProcessor` sẽ tối ưu hóa/nén ảnh.
5.  **Event Store:** Ghi nhận sự kiện `ImageWasCreated`.
6.  **Read Model Projection:** Một tiến trình chạy ngầm nhận event và cập nhật dữ liệu vào bảng `ImageView` (SQL) để phục vụ việc tìm kiếm/hiển thị nhanh sau này.

#### Luồng Hiển thị thời gian thực (Real-time Update):
1.  **Hành động:** Người dùng A comment vào một ảnh.
2.  **Server:** `CommentChangeListener` (Doctrine Listener) phát hiện thay đổi.
3.  **Publisher:** Server gửi một thông điệp qua **Mercure Hub**.
4.  **Client:** Trình duyệt của Người dùng B (đang xem ảnh đó) nhận được thông báo qua SSE và tự động cập nhật danh sách comment mà không cần load lại trang.

#### Luồng Xác thực (Authentication):
1.  **SSO/OIDC:** Hỗ trợ đăng nhập qua các bên thứ ba.
2.  **JWT:** Sau khi đăng nhập thành công, server trả về JWT (JSON Web Token).
3.  **API Key:** Hỗ trợ các ứng dụng bên ngoài (như ShareX) truy cập API thông qua khóa bí mật cố định.

### Tổng kết
**Slink** không đơn thuần là một ứng dụng PHP thông thường. Nó là một hệ thống được thiết kế với tư duy **Enterprise**, sử dụng các công nghệ tối ưu về tốc độ (libvips, FrankenPHP) và các mẫu thiết kế bền vững (Event Sourcing, CQRS). Đây là một dự án mẫu mực cho việc kết hợp giữa Symfony và Svelte để tạo ra ứng dụng Full-stack hiệu suất cao.