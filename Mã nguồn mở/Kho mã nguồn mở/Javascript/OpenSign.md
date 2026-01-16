Dựa trên nội dung file README và cấu trúc mã nguồn của dự án **OpenSign**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng trên mô hình **MERN Stack** (nhưng thay đổi chữ N từ Node thuần sang Parse Server) mang tính module hóa cao:

*   **Frontend (Client):**
    *   **React 19:** Phiên bản mới nhất của thư viện UI phổ biến.
    *   **Vite:** Công cụ build nhanh thay thế cho Create React App.
    *   **Tailwind CSS & DaisyUI:** Framework CSS để thiết kế giao diện nhanh và hỗ trợ Dark Mode mạnh mẽ.
    *   **Redux Toolkit:** Quản lý trạng thái toàn cục (User info, Sidebar, Widget status).
    *   **PDF Manipulation:** `pdf-lib` (xử lý file PDF), `react-pdf` (hiển thị PDF), `react-konva` (vẽ các lớp annotation lên PDF).
*   **Backend (Server):**
    *   **Node.js & Express:** Nền tảng thực thi chính.
    *   **Parse Server:** Một framework mã nguồn mở mạnh mẽ (MBaaS) giúp quản lý Database Schema, Auth, và File tự động thông qua API.
    *   **Digital Signing:** `@signpdf/signpdf`, `pkijs`, `node-forge` (thực hiện ký số chuẩn PKCS#7).
*   **Database & Storage:**
    *   **MongoDB:** Lưu trữ dữ liệu dạng tài liệu (NoSQL).
    *   **Adapters:** Hỗ trợ lưu trữ file tại địa phương (Local) hoặc Cloud (AWS S3, DigitalOcean Spaces).
*   **DevOps & Deployment:**
    *   **Docker & Docker-compose:** Đóng gói toàn bộ hệ thống.
    *   **Caddy:** Làm Reverse Proxy và tự động cấp chứng chỉ SSL.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Design Thinking)

*   **Kiến trúc Client-Server tách biệt:** Frontend (`apps/OpenSign`) và Backend (`apps/OpenSignServer`) hoạt động độc lập, giao tiếp qua REST API.
*   **Kiến trúc Multi-tenancy:** Hệ thống được thiết kế để hỗ trợ nhiều tổ chức (Organizations/Tenants). Mỗi người dùng gắn liền với một `TenantId`, giúp phân tách dữ liệu giữa các doanh nghiệp khác nhau.
*   **Kiến trúc hướng sự kiện (Event-driven):** Sử dụng các hàm `afterSave`, `beforeSave` của Parse Server để tự động hóa các tác vụ như gửi email thông báo sau khi tài liệu được cập nhật hoặc tạo chứng chỉ (Certificate) ngay khi tài liệu hoàn tất.
*   **Tư duy mở rộng (Scalability):** Cho phép thay thế File Adapter dễ dàng (chuyển từ Local sang S3) mà không cần sửa code cốt lõi.

---

### 3. Các kỹ thuật chính nổi bật (Technical Highlights)

*   **Xử lý PDF nâng cao trên trình duyệt:**
    *   Sử dụng **Canvas (Konva)** để tạo các lớp phủ (overlay) cho phép người dùng kéo-thả (Drag & Drop) các Widget như chữ ký, ngày tháng, văn bản vào đúng tọa độ trên PDF.
    *   Kỹ thuật tính toán tỷ lệ (scaling) để đảm bảo vị trí chữ ký khớp chính xác giữa các màn hình có độ phân giải khác nhau.
*   **Hệ thống ký số bảo mật:**
    *   Nhúng chữ ký vào PDF bằng cách tạo các `Placeholder` (vùng chờ), sau đó dùng chứng chỉ số (.pfx/.p12) để ký số thực sự, đảm bảo tính toàn vẹn của tài liệu (không thể chỉnh sửa sau khi ký).
    *   Tự động tạo **Certificate of Completion** chứa Audit Trail (nhật ký lịch sử: IP, thời gian, email) cho mỗi tài liệu.
*   **Chuyển đổi tài liệu:** Tích hợp **LibreOffice** trong Docker để chuyển đổi file `.docx` sang `.pdf` trực tiếp trên server.
*   **Xác thực 2 lớp cho khách (Guest OTP):** Người ký không cần tài khoản vẫn có thể xác thực danh tính qua mã OTP gửi tới email trước khi được phép xem và ký tài liệu.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow Summary)

Luồng hoạt động chính của một quy trình ký tài liệu trong OpenSign diễn ra như sau:

1.  **Giai đoạn Chuẩn bị (Upload & Setup):**
    *   Chủ sở hữu tải lên file PDF hoặc Docx.
    *   Hệ thống xác định các "Người ký" (Signers) và gán "Vai trò" (Roles).
    *   Chủ sở hữu kéo thả các Widget (Chữ ký, Họ tên, Ngày tháng) vào vị trí cần thiết trên tài liệu.
2.  **Giai đoạn Gửi (Dispatch):**
    *   Hệ thống lưu tài liệu dưới dạng "Draft" hoặc "In Progress".
    *   Gửi email mời ký (Invitations) theo thứ tự đã thiết lập (Sequential Signing) hoặc gửi đồng thời.
3.  **Giai đoạn Ký (Signing):**
    *   Người nhận nhấp vào link bảo mật trong email.
    *   Xác thực OTP (nếu được yêu cầu).
    *   Thực hiện ký bằng cách vẽ tay, nhập văn bản hoặc tải ảnh chữ ký lên.
4.  **Giai đoạn Hoàn tất (Finalization):**
    *   Sau khi người cuối cùng ký xong, Server tiến hành làm phẳng (flatten) tài liệu và nhúng chứng chỉ số của hệ thống.
    *   Tạo file PDF hoàn tất kèm theo Chứng chỉ hoàn thành (Completion Certificate).
    *   Tự động gửi bản sao tài liệu đã ký cho tất cả các bên qua email và lưu trữ vào **OpenSign Drive**.

Dự án này là một ví dụ điển hình về việc kết hợp các thư viện xử lý file phức tạp với một backend linh hoạt để tạo ra một ứng dụng thay thế sản phẩm thương mại một cách chuyên nghiệp.