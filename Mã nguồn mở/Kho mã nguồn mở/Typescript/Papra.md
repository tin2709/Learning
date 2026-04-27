Dựa trên cấu trúc thư mục và nội dung các file (README, tài liệu kiến trúc, cấu hình), dưới đây là phân tích chi tiết về dự án **Papra** - một nền tảng quản lý và lưu trữ tài liệu mã nguồn mở.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án được xây dựng theo mô hình **Monorepo** sử dụng `pnpm workspaces`, chia thành các ứng dụng (`apps`) và các gói thư viện dùng chung (`packages`).

*   **Frontend (Web):** 
    *   **SolidJS:** Một thư viện UI declarative nhưng không dùng Virtual DOM như React, giúp đạt hiệu suất cực cao và dung lượng nhẹ.
    *   **Shadcn Solid & UnoCSS:** Hệ thống UI component và công cụ CSS nguyên tử (atomic CSS) giúp giao diện tinh gọn, hỗ trợ Dark Mode.
*   **Backend (Server):**
    *   **HonoJS:** Web framework cực nhanh và nhẹ, chạy được trên nhiều môi trường (Node.js, Bun, Cloudflare Workers).
    *   **Drizzle ORM:** TypeScript ORM hiện đại, giúp tương tác với database một cách type-safe.
    *   **Better Auth:** Hệ thống xác thực toàn diện, hỗ trợ 2FA, SSO, và các nhà cung cấp OAuth2 tùy chỉnh.
*   **Database & Search:**
    *   **SQLite (LibSQL/Turso):** Sử dụng SQLite làm database chính để dễ dàng tự host.
    *   **FTS5 (Full-Text Search):** Tích hợp tìm kiếm toàn văn ngay trong SQLite để tìm nội dung bên trong tài liệu.
*   **Mobile:** 
    *   **Expo/React Native:** Cho phép phát triển ứng dụng di động đa nền tảng (iOS/Android) từ cùng một codebase.
*   **Xử lý tài liệu:**
    *   **Tesseract.js:** Nhận dạng ký tự quang học (OCR) để trích xuất văn bản từ ảnh và file PDF scan.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Papra tập trung vào 3 trụ cột: **Tối giản (Minimalism)**, **Tin cậy (Reliability)** và **Quyền riêng tư (Privacy)**.

*   **Nguyên tắc "No-Mutation" (Không biến đổi):** Papra cam kết không bao giờ sửa đổi file gốc của người dùng. Khi bạn tải lên file A, bạn sẽ luôn nhận lại đúng file A. Mọi quá trình xử lý như OCR hay trích xuất metadata đều được lưu riêng biệt.
*   **Cấu trúc Tổ chức (Organizations):** Hỗ trợ đa người dùng và phân quyền theo tổ chức. Tài liệu được quản lý theo không gian chung (gia đình, công ty) thay vì chỉ cá nhân.
*   **Khả năng tự host (Self-hosting centric):** Docker image cực nhẹ (<200MB), hỗ trợ nhiều kiến trúc CPU (ARM, x86), cấu hình linh hoạt qua biến môi trường hoặc file YAML/JSON.
*   **Soft Delete & Purge:** Khi xóa tổ chức, dữ liệu không mất ngay mà được đưa vào trạng thái chờ xóa trong 30 ngày (Grace period) để có thể khôi phục, sau đó mới thực hiện xóa cứng (Hard delete).

---

### 3. Các kỹ thuật then chốt (Key Techniques)

*   **Khử trùng lặp tài liệu (Deduplication):** Sử dụng thuật toán **SHA-256** để băm (hash) nội dung file. Nếu cùng một file được tải lên nhiều lần trong một tổ chức, hệ thống chỉ lưu một bản copy duy nhất để tiết kiệm bộ nhớ.
*   **Mã hóa tài liệu (Encryption):** 
    *   Sử dụng mô hình hai lớp: **KEK (Key Encryption Key)** và **DEK (Document Encryption Key)**.
    *   Mỗi tài liệu có một DEK riêng (AES-256-GCM). DEK này lại được mã hóa bởi KEK (do người dùng cấu hình). Điều này đảm bảo ngay cả khi hacker chiếm được file storage, họ cũng không thể đọc nếu không có database và khóa bí mật.
*   **Hệ thống Ingestion đa dạng:**
    *   **Email Ingestion:** Tích hợp với OwlRelay hoặc Cloudflare Email Workers để biến email thành tài liệu.
    *   **Folder Ingestion:** Tự động quét và nhập file từ một thư mục chỉ định (Watch folder).
*   **Tagging Rules (Quy tắc gắn thẻ tự động):** Cho phép người dùng thiết lập các điều kiện (ví dụ: nếu tên file chứa "Invoice" thì tự động gắn thẻ "Hóa đơn").

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động từ lúc nhập liệu đến lúc truy xuất diễn ra như sau:

1.  **Nhập liệu (Ingestion):** Người dùng đẩy file vào qua giao diện Web, App di động, gửi Email, hoặc bỏ vào thư mục Ingestion.
2.  **Xử lý sơ bộ (Processing):**
    *   Hệ thống tính toán **SHA-256 Hash** để kiểm tra trùng lặp.
    *   Nếu mã hóa được bật, file sẽ được mã hóa bằng **AES-256-GCM**.
3.  **Trích xuất nội dung (Extraction):**
    *   Sử dụng thư viện `lecture` (trong `packages/lecture`) để đọc nội dung văn bản.
    *   Nếu là ảnh hoặc PDF scan, **OCR (Tesseract)** sẽ được kích hoạt để lấy chữ.
    *   Nội dung này được đánh chỉ mục vào **SQLite FTS5**.
4.  **Phân loại tự động (Automation):** Các **Tagging Rules** được chạy để kiểm tra điều kiện và gắn thẻ (Tags) tự động cho tài liệu.
5.  **Lưu trữ (Storage):** 
    *   File vật lý được lưu ở Local FS hoặc S3/Azure Blob.
    *   Metadata (tên, hash, tags, nội dung trích xuất) được lưu vào database.
6.  **Truy xuất (Retrieval):** Người dùng tìm kiếm bằng ngôn ngữ tự nhiên (Advanced Search). Hệ thống giải mã file (nếu cần) và cung cấp link tải/xem trước.

### Kết luận
Papra là một ví dụ điển hình về việc ứng dụng **Modern TypeScript Stack** (Hono, Solid, Drizzle) để tạo ra một công cụ Productivity mạnh mẽ nhưng vẫn đảm bảo tính gọn nhẹ, dễ triển khai cho người dùng cá nhân và doanh nghiệp nhỏ.