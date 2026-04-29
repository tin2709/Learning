Dựa trên mã nguồn và tài liệu bạn cung cấp, đây là phân tích chi tiết về dự án **UnInbox** - một hạ tầng giao tiếp mã nguồn mở hiện đại.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án sử dụng các công nghệ tiên tiến nhất trong hệ sinh thái TypeScript để đảm bảo tốc độ và tính an toàn cao:

*   **Ngôn ngữ & Quản lý Repo:** TypeScript, kiến trúc **Monorepo** với `pnpm` và `Turborepo` để quản lý đa dịch vụ trong một kho mã duy nhất.
*   **Frontend:** **Next.js (App Router)**. Sử dụng **Tailwind CSS** và **Shadcn/UI** cho giao diện, **Framer Motion** cho hiệu ứng mượt mà.
*   **Backend Framework:** **Hono** (một web framework siêu nhanh) được dùng cho hầu hết các microservices (`platform`, `mail-bridge`, `storage`, `worker`).
*   **API Layer:** **tRPC** (đảm bảo Typesafe 100% từ Server sang Client) kết hợp với **Zod** để schema validation.
*   **Database & ORM:** **Drizzle ORM** (hiệu suất cao, kiểu dữ liệu chặt chẽ) kết hợp với **MySQL** (thiết kế tối ưu cho PlanetScale).
*   **Xử lý thời gian thực (Real-time):** Giao thức **Pusher** (thông qua server tự host **Soketi**).
*   **Hệ thống xác thực:** **Lucia Auth**, hỗ trợ cả Password truyền thống, **2FA (TOTP)** và đặc biệt là **Passkeys (WebAuthn)**.
*   **Nền tảng Email:** Tích hợp với **Postal** (Mail server mã nguồn mở) để xử lý việc gửi/nhận email thô.
*   **Cơ sở hạ tầng khác:** **Redis** (dùng cho BullMQ và Caching), **S3/Minio** (lưu trữ file đính kèm/avatar).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của UnInbox được xây dựng theo hướng **Dịch vụ hóa (Service-oriented)** và **Phân rã (Decoupled)**:

*   **Tách biệt trách nhiệm (Separation of Concerns):**
    *   `apps/web`: Chỉ lo về UI/UX.
    *   `apps/platform`: Đóng vai trò là "bộ não", quản lý logic nghiệp vụ (Auth, Org, Spaces).
    *   `apps/mail-bridge`: Cầu nối trung gian giữa hệ thống email thô (SMTP/Postal) và database của ứng dụng.
    *   `apps/storage`: Quản lý riêng việc xử lý ảnh, tạo Presigned URL để bảo mật file.
    *   `apps/worker`: Xử lý các tác vụ nền (Cron jobs, DNS check) để không làm nghẽn API chính.
*   **Kiến trúc hướng sự kiện (Event-driven):** Sử dụng Pusher để cập nhật UI ngay lập tức khi có email mới hoặc thay đổi trạng thái mà không cần reload trang.
*   **Hệ thống Định danh duy nhất (TypeID):** Một điểm sáng kiến trúc là việc sử dụng **TypeID** (ví dụ: `a_...` cho account, `c_...` cho convo). Điều này giúp phân biệt loại object ngay từ ID và tăng tính bảo mật, tránh nhầm lẫn dữ liệu.
*   **Thiết kế sẵn sàng cho Serverless:** Mã nguồn đang có lộ trình chuyển dịch sang **Vercel Workflow** (thay thế BullMQ) để loại bỏ các tiến trình chạy ngầm dài hạn, tối ưu hóa chi phí và khả năng mở rộng.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý Email phức tạp:**
    *   Sử dụng `mailparser` để bóc tách email MIME.
    *   Kỹ thuật **Content-ID (CID) mapping**: Chuyển đổi các ảnh nhúng trong email thô thành URL lưu trữ của hệ thống và hiển thị an toàn qua Proxy.
    *   **Sanitize HTML:** Sử dụng `DOMPurify` (với JSDOM trên backend) để làm sạch nội dung email, chống tấn công XSS khi hiển thị.
*   **Bảo mật nâng cao:**
    *   **Elevated Mode (Sudo Mode):** Khi thực hiện tác vụ nhạy cảm (đổi pass, xóa acc), người dùng phải xác thực lại dù đã đăng nhập.
    *   **Signature Verification:** Kiểm tra chữ ký webhook từ Postal để đảm bảo dữ liệu email đến từ nguồn tin cậy.
*   **Tối ưu hóa UI:**
    *   **Virtual Scrolling (`react-virtuoso`):** Hiển thị danh sách hội thoại và tin nhắn cực dài mà không làm chậm trình duyệt.
    *   **Rich Text Editor:** Tùy biến từ **Tiptap**, hỗ trợ Slash commands (`/`) giống Notion.
*   **Observability:** Tích hợp **OpenTelemetry (OTEL)** xuyên suốt các service để theo dõi vết (tracing) và hiệu năng hệ thống.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

#### A. Luồng nhận Email (Incoming Mail):
1.  **Postal Server** nhận email từ internet -> Gửi Webhook tới `mail-bridge`.
2.  `mail-bridge` xác thực chữ ký -> Đẩy job vào **BullMQ (Redis)**.
3.  **Mail Processor Worker** bóc tách email:
    *   Phân tích người gửi/người nhận để xác định thuộc về **Organization/Space** nào.
    *   Lưu file đính kèm vào **S3**.
    *   Lưu nội dung vào bảng `convo_entries`.
4.  **Real-time Server** phát tín hiệu tới Frontend -> Người dùng thấy tin nhắn mới hiện lên ngay lập tức.

#### B. Luồng gửi Email (Outgoing Mail):
1.  Người dùng soạn thảo trên Web (Next.js) -> Gửi yêu cầu qua **tRPC** tới `platform`.
2.  `platform` gọi sang `mail-bridge`.
3.  `mail-bridge` sử dụng **MailComposer** để đóng gói email và gửi qua SMTP của hệ thống hoặc SMTP bên thứ 3 (nếu người dùng cấu hình External Email).

#### C. Luồng Quản lý Tên miền (DNS Check):
1.  Người dùng thêm Domain -> `platform` tạo bản ghi DKIM/SPF.
2.  `apps/worker` định kỳ kiểm tra bản ghi DNS của Domain đó thông qua các DNS Resolver (Cloudflare).
3.  Nếu DNS hợp lệ, cập nhật trạng thái `active` để cho phép gửi/nhận mail chính thức.

---
**Kết luận:** UnInbox không chỉ là một ứng dụng email, mà là một hệ điều hành giao tiếp thu nhỏ với kiến trúc rất bài bản, chú trọng vào tính an toàn dữ liệu và khả năng mở rộng quy mô lớn.