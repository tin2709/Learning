Dưới đây là phân tích chi tiết về dự án **Postiz** (gitroomhq/postiz-app) dựa trên cấu trúc thư mục và thông tin bạn cung cấp:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một Stack hiện đại, tập trung vào hiệu suất và khả năng mở rộng:

*   **Ngôn ngữ & Quản lý Package:** 
    *   **TypeScript (73.5%):** Đảm bảo an toàn kiểu dữ liệu cho toàn bộ hệ thống.
    *   **pnpm Workspaces:** Kiến trúc Monorepo giúp quản lý nhiều package/apps trong một repository duy nhất một cách hiệu quả.
*   **Frontend:** 
    *   **Next.js (React):** Framework mạnh mẽ cho giao diện người dùng, hỗ trợ Server-side Rendering (SSR) và tối ưu SEO.
    *   **Tailwind CSS & Mantine:** Dùng để xây dựng giao diện nhanh chóng và nhất quán.
*   **Backend:**
    *   **NestJS:** Framework Node.js theo kiến trúc Module, dễ bảo trì và mở rộng.
    *   **Prisma (ORM):** Công cụ giao tiếp với Database (mặc định là PostgreSQL) thông qua Type-safe API.
*   **Điều phối Workflow (Quan trọng nhất):**
    *   **Temporal:** Đây là "trái tim" của hệ thống scheduling. Temporal đảm bảo các tác vụ (như đăng bài sau 3 ngày, gửi email nhắc nhở) hoạt động tin cậy, tự động thử lại (retry) nếu lỗi mà không mất trạng thái.
*   **Cơ sở hạ tầng & Khác:**
    *   **Redis:** Dùng cho caching và giới hạn lưu lượng (throttling).
    *   **Docker:** Hỗ trợ đóng gói ứng dụng để triển khai dễ dàng trên mọi môi trường.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Postiz được thiết kế theo hướng **Modularity (Tính module)** và **Reliability (Tính tin cậy)**:

*   **Kiến trúc Monorepo:** 
    *   `apps/`: Chứa các ứng dụng độc lập (Frontend, Backend, Orchestrator, CLI, Extension).
    *   `libraries/`: Chứa các logic dùng chung (Database, DTOs, Integrations, Shared UI). Tư duy này giúp giảm thiểu việc lặp code (DRY - Don't Repeat Yourself).
*   **Provider Pattern (Mô hình nhà cung cấp):** Hệ thống tích hợp mạng xã hội (X, Facebook, LinkedIn...) được thiết kế dưới dạng các "Providers". Mỗi mạng xã hội là một lớp (class) kế thừa từ một lớp trừu tượng (`social.abstract.ts`). Điều này cho phép dễ dàng thêm mạng xã hội mới mà không ảnh hưởng đến code lõi.
*   **Event-Driven & Workflow-Centric:** Thay vì dùng các lệnh `setTimeout` hay `cron job` truyền thống (dễ mất dữ liệu khi server sập), dự án sử dụng Temporal để quản lý vòng đời của một bài viết từ lúc lên lịch đến lúc đăng thành công.
*   **Headless API-First:** Dự án cung cấp cả Web UI, CLI và SDK, cho phép các nhà phát triển khác tích hợp Postiz vào hệ thống của họ một cách linh hoạt.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Quản lý OAuth phức tạp:** Xử lý luồng xác thực đa nền tảng (OAuth2) cho hàng chục mạng xã hội khác nhau, quản lý việc làm mới token (refresh tokens) tự động.
*   **AI Integration:** Tích hợp OpenAI, Fal.ai, Heygen để hỗ trợ tạo nội dung (văn bản, hình ảnh, video) tự động cho bài viết.
*   **Xử lý Media:** Hệ thống upload và lưu trữ hỗ trợ Cloudflare R2, Local storage và xử lý video/hình ảnh trước khi đăng.
*   **I18n (Quốc tế hóa):** Hỗ trợ đa ngôn ngữ (Việt, Anh, Pháp, Nhật...) thông qua các file JSON được quản lý tập trung trong thư mục `libraries/react-shared-libraries`.
*   **Extension & Mobile Support:** Có ứng dụng extension để hỗ trợ lấy cookie/xác thực cho các nền tảng khó (như Skool).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Một quy trình điển hình của Postiz diễn ra như sau:

1.  **Giai đoạn soạn thảo (Frontend):** Người dùng vào Next.js app, soạn nội dung, dùng AI để tạo ảnh/caption và chọn thời gian đăng bài.
2.  **Lưu trữ (Backend):** Backend (NestJS) nhận yêu cầu, kiểm tra quyền hạn (Permissions/Guard) và lưu bài viết vào PostgreSQL thông qua Prisma với trạng thái "Scheduled".
3.  **Kích hoạt Workflow (Orchestrator):** Một tín hiệu được gửi đến Temporal. Temporal sẽ tạo một "Workflow" cho bài viết đó.
4.  **Chờ đợi & Thực thi:** Khi đến thời điểm đăng bài, Temporal Worker (trong `apps/orchestrator`) sẽ thức dậy, gọi đến `post.activity.ts`.
5.  **Tương tác API mạng xã hội:** `post.activity` sẽ sử dụng `Integration Manager` để chọn đúng Provider (ví dụ `x.provider.ts`). Provider này thực hiện gọi API chính thức của nền tảng (X, LinkedIn...) để đăng bài.
6.  **Xử lý kết quả:** 
    *   Nếu thành công: Cập nhật trạng thái bài viết thành "Published".
    *   Nếu lỗi (API sập, Token hết hạn): Temporal sẽ tự động thực hiện chính sách "Retry" sau một khoảng thời gian nhất định.
7.  **Thông báo:** Sau khi hoàn tất, hệ thống gửi thông báo qua Email (Resend) hoặc thông báo trong app cho người dùng.

### Kết luận
Postiz không chỉ là một ứng dụng đăng bài đơn thuần mà là một **hệ thống điều phối (Orchestration System)** mạnh mẽ. Điểm mạnh nhất của nó nằm ở việc sử dụng **Temporal** để giải quyết bài toán "đăng bài tin cậy" và kiến trúc **Provider** giúp mở rộng không giới hạn các kênh mạng xã hội.