Dưới đây là phân tích chi tiết về dự án **Comp AI** (trycompai/comp) dựa trên mã nguồn và cấu trúc bạn cung cấp. Đây là một nền tảng tuân thủ (compliance platform) hiện đại, sử dụng mô hình "Open Core".

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Quản lý Monorepo:** Sử dụng **Turborepo** kết hợp với **Bun**. Đây là bộ đôi tối ưu cho tốc độ build và quản lý gói phụ thuộc trong hệ sinh thái TypeScript.
*   **Backend (API):** Sử dụng **NestJS** (`apps/api`). Đây là "Source of Truth" duy nhất cho logic nghiệp vụ, xác thực và phân quyền.
*   **Frontend:** Gồm 3 ứng dụng **Next.js**:
    *   `apps/app`: Dashboard quản lý tuân thủ chính.
    *   `apps/portal`: Cổng thông tin dành cho nhân viên.
    *   `apps/framework-editor`: Công cụ chỉnh sửa các bộ tiêu chuẩn (SOC2, ISO...).
*   **Cơ sở dữ liệu & ORM:** **PostgreSQL** kết hợp với **Prisma**. Cấu trúc Schema được chia nhỏ thành nhiều file trong `packages/db/prisma/schema` để dễ quản lý.
*   **Xác thực (Auth):** Sử dụng **better-auth**, tập trung tại API và dùng session-based (cookie cross-subdomain), không dùng JWT để tăng tính bảo mật và khả năng thu hồi phiên.
*   **Workflow & Automation:** **Trigger.dev** (v4) đóng vai trò then chốt trong việc thực hiện các tác vụ nền như quét bảo mật đám mây, gửi email thông báo, và thu thập minh chứng tự động.
*   **AI/LLM:** Tích hợp **AI SDK** (Vercel) hỗ trợ nhiều model (OpenAI, Anthropic, Groq). Sử dụng **Upstash Vector** để lưu trữ và tìm kiếm ngữ cảnh cho tính năng AI assistant.
*   **Cơ sở hạ tầng:** Docker, AWS (S3, ECR/ECS), Vercel.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Comp AI rất chặt chẽ và hướng đến khả năng mở rộng (scale):

*   **API-Centric:** Dự án đang chuyển dịch từ Next.js Server Actions sang việc gọi trực tiếp NestJS API. Điều này giúp tách biệt hoàn toàn giao diện và logic, cho phép các client khác (như Device Agent) dùng chung một hệ thống backend.
*   **RBAC (Phân quyền dựa trên hành động):** Hệ thống phân quyền phẳng (`resource:action`) cực kỳ chi tiết. Quyền được định nghĩa tập trung tại `packages/auth` và áp dụng đồng bộ từ database, API guards đến UI components. Quyền được kiểm tra ở mọi endpoint bằng `@RequirePermission`.
*   **Kiến trúc Đa thuê (Multi-tenancy):** Mọi truy vấn database đều bắt buộc phải có `organizationId` để đảm bảo cô lập dữ liệu giữa các khách hàng.
*   **Design System nhất quán:** Sử dụng Tailwind CSS và một thư viện thành phần nội bộ (`@trycompai/design-system`) dựa trên Carbon Icons, thay thế dần các thư viện cũ để đảm bảo trải nghiệm người dùng đồng nhất.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Tự động hóa minh chứng (Automated Evidence):** Sử dụng các "Adapters" cho AWS, Azure, GCP để quét cấu hình hạ tầng và đối chiếu với các yêu cầu tuân thủ.
*   **Tiền tố ID (Prefixed IDs):** Mọi ID trong database đều có tiền tố để dễ nhận diện (ví dụ: `org_abc`, `tsk_123`), sử dụng hàm PL/pgSQL tùy chỉnh.
*   **Prisma Extensions:** Sử dụng extension để tự động hóa các tác vụ lặp đi lặp lại hoặc can thiệp vào vòng đời truy vấn (như audit logging).
*   **AI-Native Policy Editor:** Kỹ thuật xử lý văn bản (chunking, embedding) để AI có thể gợi ý chỉnh sửa chính sách (policies) dựa trên ngữ cảnh riêng của từng công ty.
*   **Browserbase Integration:** Sử dụng Browserbase để tự động hóa việc chụp ảnh màn hình hoặc thao tác trên web làm bằng chứng tuân thủ (evidence).
*   **Audit Logging Interceptor:** Trong NestJS, một Interceptor sẽ tự động ghi lại mọi thay đổi dữ liệu vào bảng `audit_log` nếu endpoint đó có gắn decorator phân quyền.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Thiết lập (Onboarding):** Người dùng tạo tổ chức, chọn các khung tuân thủ (như SOC 2). Hệ thống tự động khởi tạo danh sách các Control và Task cần thực hiện.
2.  **Thu thập dữ liệu:**
    *   **Tự động:** API kết nối với GitHub, AWS, Vercel... thông qua Trigger.dev để chạy các "Check". Kết quả (đạt/không đạt) được đẩy về hệ thống.
    *   **Thủ công:** Nhân viên upload tài liệu lên S3 hoặc ký chính sách trên ứng dụng Portal.
3.  **Xử lý AI:** AI assistant phân quét các tài liệu đã upload, hỗ trợ điền bảng câu hỏi bảo mật (Questionnaire) hoặc viết lại các chính sách bảo mật cho phù hợp với tiêu chuẩn.
4.  **Giám sát & Khắc phục:** Dashboard hiển thị tỷ lệ tuân thủ theo thời gian thực. Nếu phát hiện vi phạm (drift), hệ thống tạo Task và gửi thông báo qua email/Slack cho người chịu trách nhiệm.
5.  **Kiểm tra (Audit):** Khi đến kỳ kiểm toán, kiểm toán viên truy cập vào "Auditor View" để xem toàn bộ minh chứng đã được hệ thống sắp xếp sẵn theo các Control tương ứng.

### Đánh giá:
Đây là một dự án có độ phức tạp cao, tổ chức mã nguồn cực kỳ chuyên nghiệp (sử dụng nhiều kỹ thuật tiên tiến của hệ sinh thái TypeScript/Next.js). Việc sử dụng monorepo giúp Comp AI duy trì sự đồng bộ giữa backend, frontend và các công cụ vệ tinh một cách hiệu quả.