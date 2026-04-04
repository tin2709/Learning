Dựa trên mã nguồn của dự án **Chronex**, đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Dự án sử dụng bộ stack hiện đại nhất (Bleeding edge) trong hệ sinh thái TypeScript:
*   **Framework:** Next.js 16 (App Router) & React 19 (phiên bản mới nhất).
*   **Communication:** **tRPC v11** - Cung cấp Type-safety tuyệt đối từ Server đến Client mà không cần định nghĩa API thủ công.
*   **Backend xử lý nền:** **Cloudflare Workers** & **Queues**. Đây là lựa chọn tối ưu về chi phí và khả năng mở rộng (Serverless).
*   **Database:** **Postgres (Neon)** kết hợp với **Drizzle ORM**. Neon hỗ trợ serverless tốt, Drizzle cung cấp tốc độ truy vấn nhanh và type-safe schema.
*   **Authentication:** **Better Auth** - Một thư viện Auth mới nổi hỗ trợ tốt cho môi trường Serverless và đa nền tảng.
*   **Storage:** **Backblaze B2** (thay vì S3 để tiết kiệm chi phí egress) dùng để lưu trữ media.
*   **UI/UX:** Tailwind CSS 4, Shadcn/UI, và Framer Motion (hiện là `motion/react`) cho hiệu ứng chuyển động.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Chronex được thiết kế theo mô hình **Monorepo (pnpm workspaces)** với sự phân tách trách nhiệm rõ ràng:

*   **Kiến trúc Phân tán (Distributed System):**
    *   **Client App:** Phụ trách UI, quản lý Workspace, xử lý OAuth flow và chuẩn bị nội dung post.
    *   **Worker App:** Đóng vai trò là "Consumer". Nó tách biệt khỏi logic của website để đảm bảo việc đăng bài không bị ảnh hưởng bởi lưu lượng truy cập web.
    *   **Shared DB Package:** Schema database được tập trung tại một nơi (`packages/db`), đảm bảo cả Client và Worker luôn đồng bộ về cấu trúc dữ liệu.

*   **Cơ chế lập lịch (Scheduling Mechanism):**
    *   Hệ thống không dùng một vòng lặp `setInterval` yếu kém. Thay vào đó, nó kết hợp giữa **Cloudflare Queues** (cho các tác vụ đăng bài ngay hoặc sắp tới) và **Cron Triggers** (quét DB mỗi 12 giờ để nạp các tác vụ mới vào hàng đợi). Điều này đảm bảo độ tin cậy cực cao.

*   **Quản lý Workspace:**
    *   Kiến trúc đa khách hàng (Multi-tenancy) ở mức logic. Dữ liệu (Token, Media, Posts) luôn được gắn với `workspaceId`, cho phép người dùng quản lý nhiều dự án/đội nhóm riêng biệt.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Type-Safe Polymorphism (Đa hình an toàn về kiểu):**
    *   Trong `createPost/page.tsx`, hệ thống xử lý nhiều nền tảng (Instagram, Discord, Slack...) bằng cách sử dụng một **Registry Pattern** (`platformFieldsRegistry`). Mỗi nền tảng có một component con riêng nhưng chia sẻ một interface chung.
*   **Zod Validation:**
    *   Sử dụng Zod không chỉ để validate form mà còn để định nghĩa schema cho metadata của từng nền tảng (ví dụ: Discord cần `embed`, Slack cần `channelId`).
*   **Hệ thống xử lý Media thông minh:**
    *   Trước khi upload, hệ thống tính toán **Media Dimensions** (rộng, cao, thời lượng video) ngay tại trình duyệt. Thông tin này được gửi kèm lên Backblaze B2 dưới dạng metadata để Worker có thể sử dụng (ví dụ: Instagram yêu cầu tỷ lệ khung hình cụ thể).
*   **Optimistic UI & State Management:**
    *   Sử dụng React Query (thông qua tRPC) để quản lý cache. Khi chuyển đổi giữa các Workspace, hệ thống sử dụng Cookie kết hợp với LocalStorage để giữ trạng thái đồng bộ giữa Client và Server (Middleware proxy).

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Kết nối (Onboarding):** Người dùng thực hiện OAuth. Token (AccessToken/RefreshToken) được mã hóa và lưu vào bảng `auth_token`.
2.  **Tạo nội dung (Composition):**
    *   Media được đẩy trực tiếp từ Browser lên Backblaze B2 (Client-side upload) để giảm tải cho Server.
    *   Người dùng viết "Unified Caption" (Caption chung), sau đó có thể ghi đè (Override) riêng cho từng nền tảng.
3.  **Hàng đợi xử lý (Queuing):**
    *   Khi nhấn "Schedule", một bản ghi `post` và các bản ghi `platform_posts` được tạo.
    *   Nếu thời gian đăng trong vòng 12h, một Message được đẩy vào **Cloudflare Queue**.
4.  **Thực thi (Execution):**
    *   Cloudflare Worker nhận Message -> Lấy Token -> Tải Media từ B2 -> Gọi API của nền tảng (Instagram Graph API, LinkedIn API, v.v.).
    *   **Retry Logic:** Nếu API nền tảng lỗi, Cloudflare Queues tự động thực hiện retry dựa trên cấu hình hàng đợi (Dead Letter Queue - DLQ).
5.  **Cập nhật (Feedback Loop):** Worker cập nhật trạng thái `success` hoặc `failed` (kèm mã lỗi) vào DB. Người dùng theo dõi kết quả thực tế tại UI "Posts History".

### Tổng kết
Chronex không chỉ là một ứng dụng CRUD đơn giản, nó là một ví dụ mẫu về việc kết hợp **Serverless kiến trúc hiện đại** để giải quyết bài toán **tác vụ nền (background jobs)** và **tích hợp bên thứ ba (OAuth/APIs)** một cách chuyên nghiệp và có khả năng mở rộng tốt.