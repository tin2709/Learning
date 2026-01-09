Dưới đây là bản phân tích chi tiết về dự án **Formbricks** (Giải pháp quản trị trải nghiệm khách hàng mã nguồn mở thay thế Qualtrics) được trình bày dưới dạng file `README.md` bằng tiếng Việt.

---

# Phân Tích Kỹ Thuật Dự Án Formbricks

## 1. Công Nghệ Cốt Lõi (Core Technologies)
Dự án được xây dựng trên một Stack hiện đại, tập trung vào hiệu năng và khả năng mở rộng:

-   **Ngôn ngữ chủ đạo:** **TypeScript (98%)** - Đảm bảo an toàn kiểu dữ liệu (type-safety) trên toàn bộ hệ thống từ Frontend đến Backend.
-   **Framework:** **Next.js (App Router)** - Tận dụng Server Components để tối ưu SEO và tốc độ tải trang, cùng Server Actions để xử lý logic phía máy chủ.
-   **Quản lý Monorepo:** **Turborepo** & **pnpm** - Giúp quản lý nhiều gói (packages) và ứng dụng (apps) trong một kho lưu trữ duy nhất, tối ưu hóa thời gian build và chia sẻ code.
-   **ORM & Database:** **Prisma** phối hợp với **PostgreSQL**. Sử dụng **pgvector** để hỗ trợ các tính năng liên quan đến vector trong tương lai.
-   **UI/UX:** **React**, **Tailwind CSS**, và **Radix UI** - Cung cấp các thành phần giao diện không có kiểu dáng mặc định (headless UI) để tùy chỉnh cao.
-   **Xác thực (Authentication):** **Auth.js (NextAuth)** - Hỗ trợ đa dạng phương thức từ Email/Password đến SSO (SAML, Google, GitHub, Azure AD).
-   **Xử lý dữ liệu & Schema:** **Zod** - Dùng để validate dữ liệu ở cả Client và Server side.
-   **Testing:** **Vitest** (Unit/Integration Test) và **Playwright** (E2E Test).
-   **Hạ tầng:** **Docker**, **Valkey/Redis** (Caching/Rate limiting), **Minio/S3** (Lưu trữ file).

## 2. Kỹ Thuật và Tư Duy Kiến Trúc (Architectural Engineering & Thinking)

### Kiến trúc Monorepo Phân Lớp
Formbricks không phải là một ứng dụng nguyên khối mà được chia nhỏ thành các thành phần có thể tái sử dụng:
-   `apps/web`: Ứng dụng chính (Dashboard, Survey Editor).
-   `packages/surveys`: Thư viện render khảo sát có thể nhúng vào các nền tảng khác.
-   `packages/database`: Quản lý schema Prisma và các bản cập nhật dữ liệu (migrations).
-   `packages/js-core`: Logic cốt lõi cho SDK để tương tác với API.
-   `packages/types`: Định nghĩa kiểu dữ liệu dùng chung toàn hệ thống.

### Tư duy "Privacy-First" (Quyền riêng tư là trên hết)
Kiến trúc được thiết kế để hỗ trợ tốt việc **Self-hosting**. Người dùng có quyền kiểm soát hoàn toàn dữ liệu của họ trên hạ tầng riêng bằng Docker hoặc Kubernetes (Helm Charts).

### Tách biệt Core và Enterprise (EE)
Dự án áp dụng mô hình kinh doanh Open-core:
-   Các tính năng cơ bản nằm trong lõi mã nguồn mở (AGPLv3).
-   Các tính năng cao cấp (SSO, Audit Logs, Role-based Access) được đóng gói riêng trong thư mục `apps/web/modules/ee`, cho phép quản lý bản quyền dễ dàng mà không làm ô nhiễm mã nguồn cốt lõi.

## 3. Các Kỹ Thuật Chính Nổi Bật (Key Standout Techniques)

1.  **Survey Editor (No-code):** Hệ thống kéo thả để xây dựng khảo sát phức tạp với logic rẽ nhánh (conditional logic) được xử lý mượt mà bằng React hook form và Zod validation.
2.  **Widget Injection:** Kỹ thuật build các gói JS dưới dạng UMD bundles (`formbricks.umd.cjs`) để khách hàng có thể nhúng khảo sát vào website bất kỳ chỉ với một dòng mã `<script>`, tương tự như Google Analytics.
3.  **Hệ thống Pipeline & Integrations:** Sử dụng kiến trúc hướng sự kiện để đẩy dữ liệu khảo sát sang các nền tảng bên thứ ba như Slack, Notion, Zapier thông qua Webhooks và API.
4.  **Cơ chế Caching Thống nhất:** Sử dụng một package `cache` riêng biệt tích hợp với Redis để tối ưu hóa truy vấn cơ sở dữ liệu và giảm độ trễ cho các API public phục vụ hàng triệu display khảo sát.
5.  **Internationalization (i18n):** Tích hợp sâu rộng việc đa ngôn ngữ cho cả giao diện quản trị và giao diện khảo sát khách hàng (sử dụng công cụ lingo.dev).

## 4. Tóm Tắt Luồng Hoạt Động (Flow Summary)

Một chu kỳ hoạt động điển hình trong Formbricks diễn ra như sau:

1.  **Thiết lập:** Người dùng đăng ký, tạo Organization và Project (Workspace).
2.  **Tạo Khảo sát:**
    -   Người dùng chọn template hoặc tạo mới từ đầu.
    -   Sử dụng Editor để thêm câu hỏi và cấu hình logic hiển thị.
    -   Thiết lập đối tượng mục tiêu (Targeting) dựa trên thuộc tính người dùng (Attributes) hoặc sự kiện (Actions).
3.  **Phân phối:**
    -   **Link Survey:** Tạo một đường dẫn duy nhất để gửi qua Email/SMS.
    -   **In-app/Website Survey:** Nhúng mã tracking vào ứng dụng. Khảo sát sẽ tự động hiển thị khi các điều kiện "Trigger" được thỏa mãn.
4.  **Thu thập & Xử lý:**
    -   Khi khách hàng trả lời, dữ liệu được đẩy về `/api/v1/client/responses`.
    -   Hệ thống kiểm tra `Rate limiting` và validate dữ liệu.
    -   Kích hoạt các `Integrations` (ví dụ: gửi thông báo vào Slack).
5.  **Phân tích:**
    -   Dữ liệu được tổng hợp thời gian thực trên Dashboard.
    -   Người dùng xem báo cáo tỷ lệ hoàn thành (Completion rate), tỷ lệ rơi rớt (Drop-off) và phân tích sâu từng câu trả lời.

---
*Tài liệu này cung cấp cái nhìn tổng quan về cách Formbricks được xây dựng để trở thành một hệ thống khảo sát mạnh mẽ và linh hoạt.*