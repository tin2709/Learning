Dựa trên các tệp tin bạn đã cung cấp, dưới đây là phân tích chi tiết về **Documenso** — giải pháp thay thế mã nguồn mở cho DocuSign.

---

### 1. Công nghệ Cốt lõi (Tech Stack)

Documenso sử dụng một bộ công nghệ hiện đại, tập trung vào hiệu suất và tính an toàn kiểu (type-safety):

*   **Ngôn ngữ:** TypeScript (100% type-safe).
*   **Framework chính:** React Router v7 (trước đây là Remix) chạy trên nền **Hono** server.
*   **Cơ sở dữ liệu:** PostgreSQL với **Prisma ORM** và **Kysely** (cho các truy vấn phức tạp).
*   **Giao diện (UI/UX):** Tailwind CSS, Radix UI và **Shadcn/UI**.
*   **API:** tRPC (cho API nội bộ và v2) và ts-rest (cho API v1).
*   **Xử lý PDF:** `@libpdf/core` để thao tác và đóng dấu kỹ thuật số lên file PDF.
*   **Xác thực:** Hỗ trợ OAuth (qua thư viện Arctic), Email/Password và **WebAuthn/Passkeys**.
*   **Quản lý Task ngầm:** Inngest hoặc Local queue (lưu trong DB).
*   **Quốc tế hóa:** Lingui.

---

### 2. Tư duy Kiến trúc (Architecture)

Hệ thống được thiết kế theo mô hình **Monorepo** quản lý bởi Turbo và npm workspaces:

*   **Apps (`/apps`):**
    *   `remix`: Ứng dụng chính (Dashboard, Signing flows).
    *   `docs`: Tài liệu hướng dẫn sử dụng Fumadocs.
*   **Packages (`/packages`):**
    *   `@documenso/lib`: Chứa logic nghiệp vụ lõi (server-only, client-only, universal).
    *   `@documenso/prisma`: Định nghĩa schema database và client.
    *   `@documenso/ui`: Thư viện thành phần giao diện dùng chung.
    *   `@documenso/signing`: Xử lý ký số (hỗ trợ file P12 cục bộ hoặc Google Cloud HSM).
*   **Strategy Pattern (Provider-based):** Documenso cho phép thay thế linh hoạt các dịch vụ hạ tầng thông qua biến môi trường (Environment Variables):
    *   **Storage:** Database (Base64) hoặc S3 (AWS, MinIO).
    *   **Signing:** Local (file .p12) hoặc Cloud (Google Cloud HSM).
    *   **Email:** SMTP, Resend, hoặc MailChannels.

---

### 3. Các Kỹ thuật & Pattern Chính

1.  **Functional Programming:** Theo `CODE_STYLE.md`, dự án ưu tiên lập trình hàm, không sử dụng Class, sử dụng Arrow functions và các pattern khai báo (declarative).
2.  **Type-Safe API:** Sử dụng tRPC giúp đồng bộ kiểu dữ liệu giữa Client và Server mà không cần code-gen phức tạp.
3.  **Zod Validation:** Mọi đầu vào từ API đến Form đều được kiểm tra chặt chẽ bằng thư viện Zod.
4.  **AI-Assisted Features:** Tích hợp Google Vertex AI để tự động phát hiện các trường (fields) và người nhận (recipients) trong tài liệu PDF.
5.  **Digital Sealing (PAdES):** Kỹ thuật đóng dấu PDF tuân thủ tiêu chuẩn ISO 32000, đảm bảo tính toàn vẹn của tài liệu sau khi ký. Nếu tài liệu bị chỉnh sửa, chữ ký sẽ mất hiệu lực.

---

### 4. Luồng Hoạt động của Hệ thống (Document Workflow)

Luồng đi của một tài liệu trong Documenso bao gồm các trạng thái: `Draft` -> `Pending` -> `Completed`/`Rejected`.

#### Bước 1: Chuẩn bị (Draft)
*   Người dùng tải lên file PDF.
*   Thêm người nhận (Recipients) với các vai trò: **Signer** (Người ký), **Approver** (Người duyệt), **Viewer** (Người xem), **Assistant** (Người hỗ trợ điền form).
*   Kéo thả các trường (Fields): Signature, Initials, Date, Text, Checkbox... vào vị trí mong muốn trên PDF.

#### Bước 2: Phân phối (Pending)
*   Hệ thống gửi email thông báo (hoặc cung cấp link trực tiếp).
*   Hỗ trợ **Sequential Signing** (ký theo thứ tự): Người thứ 2 chỉ nhận được thông báo sau khi người thứ 1 đã ký xong.

#### Bước 3: Ký tài liệu
*   Người nhận truy cập link duy nhất.
*   Thực hiện các thao tác: Vẽ chữ ký, nhập text.
*   Documenso ghi lại **Audit Log** (IP, thiết bị, thời gian).

#### Bước 4: Đóng dấu & Hoàn tất (Completed)
*   Sau khi người cuối cùng ký, một job ngầm (`seal-document`) được kích hoạt.
*   Hệ thống sử dụng **Signing Provider** đã cấu hình để tạo chữ ký số (Digital Signature) mã hóa toàn bộ file PDF.
*   Tạo file PDF cuối cùng kèm theo chứng chỉ và Audit Log.
*   Gửi bản sao hoàn tất cho tất cả các bên.

---

### 5. Khả năng Mở rộng & Tự vận hành (Self-hosting)
Documenso được thiết kế cực kỳ thân thiện với việc tự cài đặt (Self-hosting):
*   Cung cấp Docker images, hỗ trợ deploy lên Railway, Render, hoặc Kubernetes.
*   Cho phép doanh nghiệp toàn quyền kiểm soát dữ liệu (Data Residency) để tuân thủ GDPR hoặc các quy định bảo mật khắt khe.

Dự án này là một ví dụ điển hình về việc xây dựng một ứng dụng doanh nghiệp (Enterprise SaaS) hiện đại bằng TypeScript với cấu trúc mã nguồn cực kỳ sạch sẽ và có tính module hóa cao.