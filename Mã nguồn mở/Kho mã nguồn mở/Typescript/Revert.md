Dưới đây là bản tóm tắt phân tích kỹ thuật của dự án **Revert** (Open-source Unified API) dưới dạng một file README tiếng Việt chuyên sâu.

---

# Phân Tích Kỹ Thuật & Kiến Trúc Hệ Thống Revert

Revert là một nền tảng **Unified API** mã nguồn mở, đóng vai trò lớp trừu tượng (abstraction layer) trung gian giữa các ứng dụng B2B và hàng loạt API của bên thứ ba (CRMs, Ticketing, Messaging).

## 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án sử dụng stack công nghệ hiện đại, tập trung vào tính an toàn của kiểu dữ liệu (type-safety) và khả năng mở rộng:

*   **Ngôn ngữ chính:** TypeScript (96.3%) - Đảm bảo tính nhất quán của dữ liệu khi chuyển đổi giữa các schema khác nhau.
*   **Backend Framework:** Node.js với Express.
*   **Cơ sở dữ liệu & ORM:** PostgreSQL kết hợp với **Prisma**. Prisma giúp quản lý migrations và định nghĩa các quan hệ phức tạp giữa Account, Connection và App.
*   **Caching & Background Jobs:** **Redis**. Sử dụng để lưu trữ cache và quản lý các tác vụ chạy ngầm như làm mới (refresh) OAuth tokens tự động thông qua `node-cron`.
*   **API Definition & SDK Generation:** **Fern**. Đây là "trái tim" của hệ thống, cho phép định nghĩa API bằng YAML và tự động tạo ra các bộ SDK (Typescript, Python, Java) cũng như tài liệu API chất lượng cao.
*   **Quản lý Webhooks:** **Svix**. Đảm bảo việc gửi và xác thực webhook đến người dùng cuối một cách tin cậy.
*   **Cơ sở hạ tầng:** Docker & Docker Compose (hỗ trợ tự triển khai - self-hosting).

## 2. Tư Duy Kỹ Thuật & Kiến Trúc (Architectural Thinking)

### A. Mô hình Monorepo (Yarn Workspaces)
Revert tổ chức code theo cấu trúc monorepo để duy trì sự đồng bộ giữa backend và các SDK client:
*   `packages/backend`: Lõi xử lý logic, quản lý auth và chuyển đổi dữ liệu.
*   `packages/js`, `packages/react`, `packages/vue`: Các thư viện UI/Logic để nhúng vào ứng dụng khách hàng.

### B. Lớp Trừu Tượng Hóa Dữ Liệu (Unified Schema)
Thay vì bắt lập trình viên phải hiểu API của cả Salesforce và HubSpot, Revert định nghĩa một **Common Schema**. 
*   **Ví dụ:** Một `Contact` trong Revert sẽ có các trường chuẩn (firstName, lastName, email) bất kể dữ liệu gốc đến từ nguồn nào.

### C. Cơ chế Bảo mật Đa lớp
*   **Secret Management:** Các token của khách hàng được mã hóa (AES-256) trước khi lưu vào DB.
*   **OAuth Proxy:** Revert đóng vai trò là một Proxy an toàn, quản lý toàn bộ luồng luân chuyển token, giúp client không bao giờ phải tiếp xúc trực tiếp với Client Secret của bên thứ ba.

## 3. Các Kỹ Thuật Key Nổi Bật (Standout Techniques)

### 1. Quy trình Unification & Disunification
Đây là kỹ thuật quan trọng nhất của Revert:
*   **Unification (Hợp nhất):** Chuyển đổi dữ liệu thô từ API bên thứ ba (ví dụ: HubSpot JSON) thành chuẩn Revert Schema trước khi trả về cho khách hàng.
*   **Disunification (Phân rã):** Khi khách hàng gửi dữ liệu lên Revert, hệ thống sẽ "phân rã" nó ngược lại định dạng mà API bên thứ ba yêu cầu (ví dụ: chuyển đổi Revert Contact thành Salesforce Lead).

### 2. Field Mapping & Custom Schema
Revert cho phép người dùng tùy biến ánh xạ (mapping) các trường dữ liệu tùy chỉnh (custom fields) từ công cụ bên thứ ba vào hệ thống của họ thông qua giao diện UI trực quan, giải quyết bài toán "long-tail" của các API phức tạp.

### 3. Tự động hóa vòng đời Token (Token Lifecycle Management)
Hệ thống sử dụng các cron job để quét và làm mới các OAuth access token sắp hết hạn. Điều này đảm bảo tích hợp không bao giờ bị gián đoạn (graceful handling of expired permissions).

### 4. API Versioning thông qua Header
Sử dụng header `x-api-version` để quản lý các phiên bản API khác nhau, cho phép hệ thống cập nhật tính năng mới mà không làm gãy các tích hợp cũ của khách hàng.

## 4. Tóm Tắt Luồng Hoạt Động (Workflow Summary)

Luồng đi của dữ liệu trong hệ thống Revert diễn ra qua 5 bước chính:

1.  **Thiết lập (Setup):** Developer tạo App trên Revert Dashboard, cấu hình Client ID/Secret của các bên thứ ba (Salesforce, HubSpot...).
2.  **Kết nối (Connection flow):** 
    *   Người dùng cuối (End-user) sử dụng Revert SDK trên Frontend của khách hàng.
    *   Thực hiện đăng nhập OAuth. 
    *   Revert nhận code, trao đổi lấy Access/Refresh Token và lưu trữ mã hóa vào PostgreSQL.
3.  **Yêu cầu dữ liệu (Request):** 
    *   Ứng dụng của khách hàng gọi API Revert (ví dụ: `GET /crm/contacts`) kèm theo `x-revert-api-token`.
4.  **Xử lý trung gian (Processing):**
    *   Backend Revert xác thực Token.
    *   Lấy Access Token tương ứng của bên thứ ba từ DB.
    *   Thực hiện gọi API thực tế đến bên thứ ba (HubSpot/Salesforce).
5.  **Chuyển đổi & Trả về (Transform & Respond):**
    *   Dữ liệu thô nhận về đi qua bộ lọc **Unify**.
    *   Revert trả về dữ liệu đã được chuẩn hóa theo Revert Schema cho ứng dụng khách hàng.

---

**Kết luận:** Revert không chỉ là một bộ chuyển đổi API đơn giản mà là một hệ thống quản lý tích hợp toàn diện, tập trung vào việc giảm thiểu "Engineering Debt" cho các công ty SaaS khi phải đối mặt với bài toán mở rộng tích hợp.