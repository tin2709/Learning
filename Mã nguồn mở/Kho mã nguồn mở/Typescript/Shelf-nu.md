Dựa trên cấu trúc thư mục và tài liệu kỹ thuật của dự án **Shelf.nu**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng quản lý tài sản mã nguồn mở này:

### 1. Công nghệ cốt lõi (Core Technology)
Shelf.nu được xây dựng trên một ngăn xếp công nghệ (stack) hiện đại, tập trung vào hiệu suất và tính nhất quán:
*   **Framework chính:** **React Router 7 (Remix v3)** kết hợp với **React 19**. Đây là kiến trúc Full-stack cho phép render phía server (SSR) và xử lý dữ liệu qua Loaders/Actions.
*   **Ngôn ngữ:** **TypeScript (99.1%)** được sử dụng xuyên suốt từ frontend đến backend để đảm bảo an toàn về kiểu.
*   **Cơ sở dữ liệu:** **PostgreSQL** (được cung cấp bởi **Supabase**). Dự án tận dụng các tính năng nâng cao như Row Level Security (RLS) và Full-text search.
*   **ORM:** **Prisma 6**, giúp quản lý schema và thực hiện các truy vấn type-safe.
*   **Quản lý State:** **Jotai** (Atomic state management) được dùng cho các trạng thái phức tạp ở client (như giỏ hàng quét QR).
*   **Giao diện:** **Tailwind CSS 3** và **Radix UI Primitives** (headless components) giúp xây dựng UI tùy biến cao và đạt chuẩn A11y (truy cập cho người khuyết tật).
*   **Hàng đợi & Tác vụ:** **pg-boss** chạy trên nền PostgreSQL để xử lý các job không đồng bộ (gửi email, tính toán lịch trình).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Dự án áp dụng mô hình **Monorepo** quản lý bởi **pnpm workspaces** và **Turborepo**:
*   **Tách biệt lớp Dữ liệu:** Thư mục `packages/database` là "nguồn sự thật duy nhất" về schema. Mọi ứng dụng khác (`webapp`, `docs`) đều tiêu thụ Prisma Client từ đây.
*   **Module-based Business Logic:** Trong ứng dụng web, logic nghiệp vụ không nằm rải rác mà tập trung vào thư mục `app/modules/`. Mỗi module (Asset, Booking, Kit, Audit) chứa service, kiểu dữ liệu và helper riêng.
*   **Kiến trúc Đa thuê (Multi-tenancy):** Hệ thống được thiết kế theo mô hình `Organization`. Dữ liệu được cách ly nghiêm ngặt dựa trên `organizationId`.
*   **Quản lý Route phẳng (Flat Routes):** Sử dụng `remix-flat-routes` để tổ chức các tệp route theo nhóm chức năng (ví dụ: `_layout+/`, `api+/`), giúp dễ bảo trì hơn cấu trúc thư mục lồng nhau truyền thống.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Select All Pattern (ALL_SELECTED_KEY):** Một kỹ thuật đặc biệt để xử lý các thao tác hàng loạt (Bulk Operations) trên tập dữ liệu lớn đã được lọc và phân trang. Thay vì gửi hàng nghìn ID từ client, hệ thống gửi một key định danh và các tham số lọc để server tự giải quyết danh sách ID.
*   **Hệ thống Quét & Xử lý Mã vạch:** Tích hợp sâu `zxing-wasm` cho việc quét mã qua camera và `bwip-js` để tạo mã vạch. Hệ thống có lớp chuẩn hóa (Normalization) để đồng bộ dữ liệu mã vạch (ví dụ: luôn viết hoa).
*   **Thuật toán Conflict Detection (Phát hiện xung đột):** Sử dụng logic SQL overlap để kiểm tra tính khả dụng của tài sản trong các khoảng thời gian đặt trước (Bookings), đảm bảo không xảy ra tình trạng "over-booking".
*   **Database Triggers & Migrations:** Sử dụng Trigger ở mức database (ví dụ: tự động tạo `UserContact` khi có `User` mới) để đảm bảo tính toàn vẹn dữ liệu mà không phụ thuộc hoàn toàn vào logic ứng dụng.
*   **Standardized Error Handling:** Sử dụng lớp `ShelfError` tùy chỉnh để phân loại lỗi (lỗi dữ liệu, lỗi hạ tầng, lỗi kết nối tạm thời) và tự động tích hợp với Sentry.

### 4. Tóm tắt luồng hoạt động (Activity Flow Summary)
Luồng hoạt động của Shelf.nu xoay quanh vòng đời của tài sản vật lý:

1.  **Thiết lập & Nhập liệu:** Người dùng tạo Organization -> Tạo Vị trí (Location - theo cấu trúc cây) -> Nhập tài sản (Asset) thủ công hoặc qua CSV.
2.  **Định danh:** Hệ thống cấp phát mã QR hoặc liên kết mã vạch hiện có cho tài sản. Người dùng in nhãn QR để dán lên thiết bị vật lý.
3.  **Vận hành (Scan-to-Action):**
    *   Người dùng dùng điện thoại quét mã QR.
    *   Hệ thống nhận diện tài sản và hiển thị Dashboard hành động nhanh (Check-out, Assign Custody, Report Found).
    *   Các thay đổi được ghi lại vào **Audit Trail** (lịch sử hoạt động).
4.  **Quản lý Đặt chỗ (Booking):** Người dùng chọn danh sách tài sản -> Chọn khoảng thời gian -> Hệ thống kiểm tra xung đột -> Tạo yêu cầu đặt chỗ (Reserved) -> Chuyển sang trạng thái Đang sử dụng (Ongoing) khi xuất kho.
5.  **Kiểm kê (Audit):** Tạo phiên kiểm kê cho một vị trí -> Nhân viên đi quét thực tế -> Hệ thống đối soát giữa số lượng "kỳ vọng" và số lượng "thực tế" quét được, sau đó xuất báo cáo PDF.