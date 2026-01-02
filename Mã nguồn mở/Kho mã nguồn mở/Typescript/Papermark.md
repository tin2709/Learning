Dưới đây là bản phân tích chi tiết về dự án **Papermark** - nền tảng chia sẻ tài liệu nguồn mở thay thế cho DocSend.

---

### 1. Công nghệ cốt lõi (Core Stack)
Dự án được xây dựng trên một hệ sinh thái JavaScript/TypeScript hiện đại, tối ưu cho hiệu năng và khả năng mở rộng:

*   **Framework:** **Next.js (App Router & Pages Router)**. Papermark sử dụng cả hai kiến trúc để tận dụng tối đa Server Components và khả năng render linh hoạt.
*   **Ngôn ngữ:** **TypeScript (99.5%)** đảm bảo tính an toàn về kiểu dữ liệu trên toàn bộ hệ thống.
*   **Cơ sở dữ liệu & ORM:** **PostgreSQL** đi kèm với **Prisma**. Đặc biệt, schema được chia nhỏ thành nhiều file (Multi-file schema) trong thư mục `prisma/schema/` giúp quản lý dự án lớn dễ dàng hơn.
*   **Phân tích dữ liệu (Analytics):** **Tinybird**. Đây là điểm khác biệt lớn, sử dụng cơ sở dữ liệu ClickHouse bên dưới để xử lý hàng triệu sự kiện (view, click, scroll) theo thời gian thực mà không làm chậm DB chính.
*   **Xử lý nền & Queue:** **Trigger.dev (v3)** và **Upstash QStash**. Dùng để xử lý các tác vụ nặng như convert PDF, gửi email hàng loạt hoặc xử lý AI.
*   **Lưu trữ (Storage):** Hỗ trợ đa nền tảng qua **AWS S3** hoặc **Vercel Blob**.
*   **AI:** Sử dụng **Vercel AI SDK** kết hợp OpenAI/Google Vertex để tạo các AI Agent phân tích tài liệu ngay trong Data Room.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

*   **Kiến trúc Core & EE (Enterprise Edition):** Dự án tách biệt rõ ràng phần mã nguồn mở (AGPLv3) và phần tính năng nâng cao (Enterprise) nằm trong thư mục `/ee` và `/app/(ee)`. Điều này cho phép họ kinh doanh mô hình "Open Core".
*   **Multi-tenancy (Đa người dùng):** Hệ thống xoay quanh khái niệm **"Team"**. Tất cả tài liệu, liên kết và tên miền tùy chỉnh đều được gắn với `teamId`, cho phép phân quyền và quản lý tài nguyên độc lập giữa các tổ chức.
*   **Middleware-centric Routing:** `middleware.ts` đóng vai trò là "trái tim" điều phối. Nó xử lý:
    *   **Custom Domains:** Nhận diện khách truy cập từ tên miền riêng của người dùng và trỏ về đúng tài liệu.
    *   **Bot Protection:** Chặn các bot hoặc các đường dẫn không hợp lệ.
    *   **PostHog/Analytics Ingest:** Chuyển hướng dữ liệu phân tích mà không bị chặn bởi các trình chặn quảng cáo.
*   **Data Room Concept:** Thay vì chỉ chia sẻ 1 file đơn lẻ, kiến trúc hỗ trợ "phòng dữ liệu" (Data Room) - nơi tập hợp nhiều folder/file với cơ chế phân quyền (Permissions) phức tạp cho từng nhóm người xem (Groups).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý tài liệu nâng cao:**
    *   Sử dụng **MuPDF** (chạy trên WebAssembly/Wasm) để render và chú thích (annotate) tài liệu PDF ngay trên trình duyệt.
    *   Sử dụng **SheetJS (XLSX)** để xem các file Excel mà không cần tải về.
    *   Kỹ thuật **PDF-to-Image conversion** để tăng tốc độ xem trang và bảo mật (ngăn copy text trực tiếp nếu cần).
*   **Bảo mật tài liệu:**
    *   **OTP (One-Time Password):** Xác thực qua email trước khi xem.
    *   **Watermarking:** Chèn watermark động (email, IP của người xem) vào tài liệu để chống rò rỉ.
    *   **Screen Protection:** Kỹ thuật ngăn chặn chụp ảnh màn hình hoặc quay phim tài liệu (EE feature).
*   **Tracking chi tiết:** Theo dõi thời gian xem từng trang (page-by-page duration), thiết bị, vị trí địa lý thông qua Tinybird Pipes.
*   **Workflow Engine:** Một bộ máy thực thi logic cho phép thiết lập luồng truy cập: ví dụ, người xem từ domain `google.com` thì dẫn vào Folder A, các domain khác dẫn vào Folder B.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Giai đoạn Upload:** Người dùng tải tài liệu lên -> Hệ thống đẩy vào S3/Vercel Blob -> Trigger.dev bắt đầu các job: tạo thumbnail, tách trang (nếu là PDF), hoặc xử lý vector để AI có thể đọc (embedding).
2.  **Giai đoạn Cấu hình:** Chủ sở hữu tạo **Link**. Tại đây họ thiết lập:
    *   Quyền truy cập (Email, mật khẩu, whitelist domain).
    *   Tên miền (Sử dụng domain mặc định của Papermark hoặc domain riêng).
    *   Tùy chỉnh thương hiệu (Logo, màu sắc).
3.  **Giai đoạn Truy cập:**
    *   Khách truy cập nhấn vào link -> Middleware kiểm tra domain/slug.
    *   Nếu link có bảo vệ -> Gửi OTP qua Resend -> Người dùng xác thực.
    *   Hệ thống tạo một **Session** và ghi nhận sự kiện "View" vào Tinybird.
4.  **Giai đoạn Trình diễn:** Trình xem (Viewer) tải dữ liệu trang tài liệu. Nếu là video, hệ thống theo dõi cả tiến trình xem (play/pause/seek).
5.  **Giai đoạn Phân tích:** Chủ sở hữu truy cập Dashboard -> Next.js gọi API đến Tinybird -> Hiển thị biểu đồ trực quan về hành vi của người xem (ai xem trang nào lâu nhất, từ đâu, khi nào).

---
**Kết luận:** Papermark là một dự án có kỹ thuật rất "chắc tay", kết hợp khéo léo giữa tính năng Product (chia sẻ file) và hạ tầng Data (Big Data analytics) cùng AI, tạo nên một giải pháp hoàn chỉnh và chuyên nghiệp.