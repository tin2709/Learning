Dựa trên cấu trúc thư mục và nội dung mã nguồn bạn cung cấp, đây là phân tích chi tiết về dự án **Logto** - một nền tảng hạ tầng định danh (Identity Infrastructure) hiện đại, mã nguồn mở, được thiết kế chuyên biệt cho các ứng dụng SaaS và AI.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này là một hệ thống cực kỳ phức tạp và quy mô lớn, sử dụng mô hình **Monorepo** (quản lý nhiều gói trong một kho lưu trữ) với các công nghệ hàng đầu:

*   **Ngôn ngữ chủ đạo:** **TypeScript (chiếm 97.1%)**, đảm bảo tính chặt chẽ và an toàn kiểu dữ liệu cho một hệ thống bảo mật.
*   **Backend (Core):**
    *   **Node.js & Koa:** Sử dụng Koa Framework (thể hiện qua các middleware `koa-*`) để xử lý API và các luồng giao thức.
    *   **PostgreSQL:** Cơ sở dữ liệu chính, sử dụng cơ chế **Row Level Security (RLS)** để hỗ trợ đa thâm canh (multi-tenancy) một cách an toàn.
    *   **Redis:** Dùng làm lớp đệm (cache) và quản lý session/well-known configs.
    *   **OIDC-provider:** Thư viện nền tảng để triển khai các giao thức chuẩn OIDC và OAuth 2.1.
*   **Frontend (Console, Experience, Account):**
    *   **React:** Thư viện chính để xây dựng giao diện người dùng.
    *   **Vite:** Công cụ build cực nhanh cho các ứng dụng frontend.
    *   **i18next:** Quản lý đa ngôn ngữ (phrases) quy mô lớn.
    *   **SCSS (CSS Modules):** Xử lý giao diện một cách hệ thống.
*   **Hạ tầng & Công cụ:**
    *   **Docker & Docker Compose:** Hỗ trợ triển khai nhanh và môi trường test tích hợp.
    *   **pnpm:** Quản lý gói trong monorepo hiệu quả.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Logto được xây dựng theo tư duy **"Identity-as-a-Service" (IDaaS)** nhưng có thể tự lưu trữ (self-hosted):

*   **Phân rã chức năng (Decoupled Architect):**
    *   `packages/core`: Nhân xử lý chính (Auth engine).
    *   `packages/console`: Giao diện quản trị cho Developer (Admin UI).
    *   `packages/experience`: Giao diện đăng nhập/đăng ký cho người dùng cuối (End-user UI).
    *   `packages/account`: Trang quản lý tài khoản cá nhân của người dùng.
    *   `packages/connectors`: Hệ thống plugin cho các dịch vụ bên thứ ba (SMS, Email, Social Login như Google, Facebook, WeChat).
*   **Thiết kế dựa trên Giao thức (Protocol-driven):** Mọi thứ xoay quanh các tiêu chuẩn công nghiệp như OpenID Connect (OIDC), OAuth 2.1 và SAML. Điều này cho phép Logto tích hợp được với mọi nền tảng khác.
*   **Đa thâm canh (Multi-tenancy):** Hỗ trợ nhiều "Tenant" (tổ chức/khách hàng) trên cùng một bộ cài đặt, với dữ liệu được cô lập hoàn toàn ở cấp độ cơ sở dữ liệu.

---

### 3. Các kỹ thuật chính (Key Technical Implementation)

*   **Hệ thống Alterations (Database Migrations):** Logto có một hệ thống quản lý thay đổi DB cực kỳ chi tiết (trong `packages/schemas/alterations`), giúp nâng cấp hệ thống mà không làm gián đoạn dữ liệu cũ.
*   **Hệ thống Connector linh hoạt:** Cho phép mở rộng khả năng gửi mã OTP hoặc đăng nhập mạng xã hội thông qua các adapter (như `connector-aliyun-sms`, `connector-google`, v.v.).
*   **WebAuthn & Passkeys:** Triển khai phương thức đăng nhập không mật khẩu hiện đại nhất hiện nay.
*   **RBAC (Role-Based Access Control):** Hệ thống phân quyền dựa trên vai trò cực kỳ chi tiết, hỗ trợ cả cấp độ ứng dụng và cấp độ tổ chức (Organization).
*   **Custom JWT:** Cho phép nhà phát triển tùy biến nội dung của Token (JWT Customizer) bằng cách chạy các đoạn mã script an toàn (Sandbox).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

#### A. Luồng Người dùng cuối (End-user Flow):
1.  **Truy cập:** Người dùng từ ứng dụng khách (Client App) được chuyển hướng đến `packages/experience`.
2.  **Định danh (Identification):** Nhập Email/Phone/Username hoặc chọn Social Login.
3.  **Xác thực (Authentication):** Nhập mật khẩu hoặc mã OTP (do các connectors xử lý).
4.  **Tương tác (Interaction):** Kiểm tra MFA (nếu bật), yêu cầu cập nhật thông tin còn thiếu.
5.  **Chấp thuận (Consent):** Hiển thị các quyền (scopes) mà ứng dụng yêu cầu.
6.  **Hoàn tất:** Trả về Authorization Code/Token cho ứng dụng khách.

#### B. Luồng Quản trị (Admin Flow):
1.  **Thiết lập:** Admin sử dụng `packages/console` để tạo ứng dụng mới (SPA, Native, hoặc M2M).
2.  **Cấu hình:** Thiết lập các connectors (gửi email, sms) và tùy chỉnh giao diện (branding).
3.  **Quản lý:** Theo dõi Audit Logs (nhật ký hệ thống), quản lý người dùng và phân quyền RBAC.
4.  **Tổ chức:** Nếu là ứng dụng B2B, Admin tạo các Organizations và mời thành viên tham gia.

**Kết luận:** Logto không chỉ là một module đăng nhập đơn giản mà là một **"Auth Infrastructure"** hoàn chỉnh, có độ phức tạp tương đương với các dịch vụ thương mại như Auth0 hay Okta, nhưng mang lại quyền kiểm soát hoàn toàn cho nhà phát triển.