Dựa trên mã nguồn của **Plunk** (nền tảng email mã nguồn mở xây dựng trên AWS SES), dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Plunk sử dụng một "stack" công nghệ hiện đại, ưu tiên tính mở rộng và hiệu suất cao:

*   **Runtime & Language:** **Node.js (v20+)** và **TypeScript**. Sử dụng TypeScript nghiêm ngặt (Strict mode) trên toàn bộ monorepo.
*   **Backend Framework:** **Express.js** nhưng được cấu trúc lại bằng `@overnightjs/core`, cho phép sử dụng Decorators (như `@Controller`, `@Post`) giống như NestJS hoặc Spring Boot.
*   **Database & ORM:** **PostgreSQL** là cơ sở dữ liệu chính, sử dụng **Prisma ORM** để quản lý Schema và Type-safe queries.
*   **Queue & Background Jobs:** **BullMQ** chạy trên nền **Redis**. Đây là thành phần quan trọng nhất để xử lý gửi email hàng loạt, chiến dịch (campaign) và tự động hóa (workflow) mà không làm nghẽn API.
*   **Email Engine:** **AWS SES** (Simple Email Service) là hạ tầng gửi thư chính. Sử dụng **MJML** để render email responsive và **LiquidJS** để xử lý template/biến số.
*   **Frontend Stack:** **Next.js 15 (App & Pages Router)**, **React 19**, **Tailwind CSS**, và **Shadcn UI**.
*   **Monorepo Management:** **Turborepo** kết hợp với **Yarn Workspaces**.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Plunk được thiết kế cho bài toán **"High Scale"** (xử lý hàng triệu contact):

*   **Tách biệt Server và Worker:** Hệ thống chia làm hai tiến trình riêng biệt:
    *   *API Server:* Chỉ tiếp nhận Request, validate dữ liệu và đẩy task vào hàng đợi.
    *   *Worker Process:* Chuyên xử lý các tác vụ nặng (gửi mail, tính toán segment) từ Redis. Điều này cho phép scale riêng lẻ các Worker khi lượng thư gửi tăng đột biến.
*   **Kiến trúc Đa thuê (Multi-tenancy):** Quản lý theo cấu trúc `User -> Membership -> Project`. Dữ liệu được phân tách theo `projectId`.
*   **Database-First cho hiệu năng:** Plunk ưu tiên sử dụng **Cursor-based pagination** thay vì Offset-based để đảm bảo khi danh sách contact lên đến hàng triệu thì tốc độ truy vấn vẫn không đổi.
*   **Shared Packages:** Các logic dùng chung (DB schema, types, UI components, email templates) được đóng gói thành các package riêng trong thư mục `packages/`, giúp đồng nhất dữ liệu giữa API và Web Dashboard.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Schema Validation:** Sử dụng **Zod** để validate mọi dữ liệu đầu vào tại Controller. Điều này đảm bảo dữ liệu trong DB luôn sạch và an toàn.
*   **Decorator-based Controllers:** Tận dụng `@overnightjs/core` để viết code backend sạch sẽ, dễ đọc và dễ bảo trì theo hướng hướng đối tượng (OOP).
*   **Content Negotiation:** Middleware xử lý thông minh việc chấp nhận các định dạng nội dung khác nhau (Markdown, HTML, JSON).
*   **SSRF Protection:** Triển khai phương thức `safeFetch` trong xử lý Webhook để ngăn chặn các cuộc tấn công Server-Side Request Forgery.
*   **Optimized Analytics Logging:** Khi một email được mở hoặc link được click, hệ thống thực hiện Redirect ngay lập tức (302) và đẩy việc ghi log vào Queue để xử lý bất đồng bộ, tối ưu trải nghiệm người dùng cuối.

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống vận hành thông qua các luồng dữ liệu khép kín:

1.  **Luồng Gửi thư Transactional:**
    *   Client gọi API `/v1/send` -> Controller validate bằng Zod -> Đẩy task vào `email-processor` (BullMQ) -> Worker lấy task -> Gọi AWS SES SDK -> Cập nhật trạng thái vào Postgres.
2.  **Luồng Chiến dịch (Campaign):**
    *   User tạo Campaign -> Chọn Segment -> Hệ thống tính toán danh sách người nhận bất đồng bộ -> Chia nhỏ thành các Batch -> Worker thực hiện gửi tuần tự theo Rate Limit của AWS SES.
3.  **Luồng Tự động hóa (Workflow):**
    *   Một sự kiện (Event) xảy ra (ví dụ: `contact.created`) -> Trigger Workflow -> Evaluator kiểm tra các điều kiện (Conditions) -> Nếu thỏa mãn, thực hiện hành động (Action) như gửi mail sau một khoảng Delay.
4.  **Luồng Theo dõi (Tracking):**
    *   AWS SES nhận Bounce/Complaint -> SNS bắn Webhook về Plunk API -> `SecurityService` xác thực chữ ký SNS -> Cập nhật trạng thái Contact (Unsubscribe hoặc Bounced) -> Tự động vô hiệu hóa Project nếu tỷ lệ lỗi vượt ngưỡng an toàn.

**Tổng kết:** Plunk là một hệ thống được thiết kế rất bài bản, tập trung vào khả năng xử lý dữ liệu lớn (Big Data) và độ tin cậy cao của hạ tầng AWS, đồng thời cung cấp một môi trường phát triển (DX) hiện đại cho lập trình viên.