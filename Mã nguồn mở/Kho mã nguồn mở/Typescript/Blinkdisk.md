Dưới đây là bản phân tích chi tiết dự án **BlinkDisk** dựa trên cấu trúc thư mục và mã nguồn bạn cung cấp:

---

### 1. Công nghệ cốt lõi (Core Stack)

Dự án được xây dựng theo mô hình **Monorepo** sử dụng **Turborepo** và **pnpm workspaces**, kết hợp nhiều ngôn ngữ và nền tảng:

*   **Desktop App:**
    *   **Framework:** Electron (Main process) kết hợp với React (Renderer process).
    *   **UI/UX:** Vite, Tailwind CSS, Lucide Icons, Framer Motion (hiệu ứng).
    *   **State Management:** TanStack Query (Server state), TanStack Router (Điều hướng), SignalDB (Database nội bộ đồng bộ trạng thái).
*   **Backup Engine (Core):** 
    *   Dựa trên **Kopia** (viết bằng **Go**), một công cụ backup mã nguồn mở mạnh mẽ xử lý mã hóa, nén và chống trùng lặp (deduplication).
*   **Backend (Sync & Auth):**
    *   **Runtime:** Cloudflare Workers (Edge Computing).
    *   **Web Framework:** Hono kết hợp với tRPC để đảm bảo type-safety từ Backend sang Frontend.
    *   **Auth:** Better-Auth (hỗ trợ cả môi trường web và Electron).
*   **Storage & Managed Cloud:**
    *   **Managed Storage (CloudBlink):** Sử dụng Cloudflare Durable Objects để quản lý trạng thái và hạn mức dung lượng (quota) theo thời gian thực.
    *   **Storage Protocols:** S3, WebDAV, SFTP, Backblaze B2, Google Cloud, Azure Blob, Rclone.
*   **Database:** PostgreSQL (truy vấn qua Kysely và Prisma).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của BlinkDisk tập trung vào sự cân bằng giữa tính riêng tư của người dùng và sự tiện lợi của đám mây:

*   **Offline-First & Security-First:** Toàn bộ quá trình mã hóa (E2EE) và băm dữ liệu (hashing) diễn ra tại máy người dùng (Client-side). Máy chủ không bao giờ nhìn thấy dữ liệu thô hoặc mật khẩu.
*   **Hybrid Storage Strategy:** Cho phép người dùng chọn giữa "CloudBlink" (dễ sử dụng, trả phí theo gói) và "Custom Storage" (tự quản lý, miễn phí phần mềm).
*   **Distributed Backend:** Sử dụng Cloudflare Workers và Hyperdrive để đưa logic backend ra sát người dùng (Edge), giảm độ trễ khi đồng bộ cấu hình.
*   **Abstraction Layer:** Ứng dụng bọc (wrap) các lệnh phức tạp của Kopia thành một giao diện đơn giản, giúp người dùng không chuyên cũng có thể thực hiện 3-2-1 backup rule.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Content-Addressable Storage (CAS):** Kỹ thuật băm nội dung file thành các block. Nếu nhiều file có nội dung giống nhau, hệ thống chỉ lưu một lần (Deduplication).
*   **Durable Objects for Quota Management:** Trong gói CloudBlink, hệ thống sử dụng Durable Objects để theo dõi dung lượng đã dùng (`usedBytes`) một cách chính xác và nhất quán, ngay cả khi có hàng nghìn yêu cầu đồng thời.
*   **tRPC & Zod Integration:** Đảm bảo dữ liệu trao đổi giữa ứng dụng Desktop và API luôn khớp kiểu dữ liệu (Type-safe), giảm thiểu lỗi runtime.
*   **WebSocket Streaming:** Ứng dụng Cloud sử dụng WebSockets để giao tiếp với Client khi thực hiện các tác vụ đẩy/kéo dữ liệu từ S3, tối ưu hiệu năng truyền tải.
*   **Policy-based Management:** Cho phép cấu hình linh hoạt về tần suất backup (Cron), thời gian lưu trữ (Retention) và thuật toán nén cho từng folder riêng biệt.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Thiết lập:** Người dùng đăng nhập (qua Better-Auth) và tạo một "Vault" (kho lưu trữ). Vault này được cấu hình trỏ đến CloudBlink hoặc S3 cá nhân.
2.  **Cấu hình Folder:** Người dùng chọn các thư mục cần bảo vệ trên máy tính. Hệ thống gán các policy (lịch trình, loại trừ file).
3.  **Thực thi Backup (Core Engine):**
    *   Engine quét thư mục, chia nhỏ file thành các chunks.
    *   **Deduplication:** Kiểm tra xem chunks đã tồn tại trên đích chưa.
    *   **Encryption:** Mã hóa chunks bằng khóa dẫn xuất từ mật khẩu Vault.
    *   **Upload:** Đẩy dữ liệu đã mã hóa lên Storage thông qua API hoặc trực tiếp S3.
4.  **Đồng bộ Metadata:** Thông tin về các bản backup (Snapshot), dung lượng đã dùng, và trạng thái Vault được đồng bộ lên API (Postgres) để người dùng có thể thấy trạng thái trên nhiều thiết bị khác nhau.
5.  **Giám sát & Cảnh báo:** Backend kiểm tra hạn mức dung lượng. Nếu sắp đầy, hệ thống gửi email cảnh báo (qua Cloudflare Workers Scheduled Events).
6.  **Khôi phục:** Người dùng chọn một thời điểm trong quá khứ (Timeline), engine sẽ tải các chunks tương ứng, giải mã và ghép lại thành file gốc.

---
**BlinkDisk** là một ví dụ điển hình của việc hiện đại hóa một công cụ dòng lệnh (CLI) mạnh mẽ (Kopia) thành một sản phẩm tiêu dùng (Consumer Product) hoàn chỉnh với hệ sinh thái Cloud hiện đại.