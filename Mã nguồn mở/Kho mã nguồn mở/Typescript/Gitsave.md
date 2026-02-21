Dựa trên mã nguồn của dự án **GitSave** mà bạn đã cung cấp, dưới đây là phân tích chi tiết về kiến trúc, công nghệ và tư duy lập trình của dự án này:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Dự án sử dụng mô hình **Full-stack Monolith** hiện đại với sự kết hợp giữa hiệu năng của Astro và sự linh hoạt của React/Node.js.

*   **Frontend Framework:** `Astro 5.0` (Sử dụng kiến trúc Islands Architecture). Astro xử lý các trang tĩnh và Server-side Rendering, giúp tốc độ tải trang cực nhanh.
*   **UI Library:** `React 18` được tích hợp vào Astro để xử lý các logic phức tạp tại Client (Forms, Popups, Dashboard logic).
*   **Styling:** `Tailwind CSS` để thiết kế giao diện responsive và dark mode mặc định.
*   **Backend:** `Express.js` chạy song song hoặc tích hợp để xử lý các API nghiệp vụ.
*   **Database & ORM:** `Prisma` kết hợp với `SQLite`. SQLite là lựa chọn thông minh cho một công cụ tự vận hành (self-hosted) vì không cần cài đặt database server phức tạp.
*   **Task Scheduling:** `node-cron` dùng để lập lịch chạy backup theo định kỳ (phút, giờ, ngày).
*   **Security:** `jsonwebtoken (JWT)` cho authentication và `bcryptjs` để hash mật khẩu. `AES-256-CBC` (thông qua module `crypto`) để mã hóa các Access Token của Git.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture Design)

Dự án áp dụng tư duy **Service-oriented Controller Pattern** ở backend, giúp mã nguồn rất sạch sẽ và dễ bảo trì:

*   **Tách biệt tầng xử lý (Layered Architecture):**
    *   **Routes:** Định nghĩa các endpoint API.
    *   **Controllers:** Tiếp nhận Request, điều phối dữ liệu (ví dụ: `schedule.controller.ts`).
    *   **Services:** Chứa logic nghiệp vụ thực tế (ví dụ: `encryption.service.ts`, `schedule.service.ts`). Đây là nơi xử lý các phép toán hoặc tương tác database phức tạp.
    *   **Models (Zod):** Sử dụng `Zod` để validate dữ liệu đầu vào ngay tại tầng controller, đảm bảo tính toàn vẹn dữ liệu trước khi xử lý.
*   **Tư duy Docker-first:** Mọi thứ được gói gọn trong Docker, sử dụng `Alpine Linux` để tối ưu dung lượng image. Việc khởi tạo database (`prisma db push`) được thực hiện ngay trong lệnh `CMD` của Dockerfile, giúp người dùng cuối chỉ cần "Plug and Play".

---

### 3. Các kỹ thuật chính nổi bật (Key Engineering Features)

#### a. Quản lý Git thông qua tiến trình con (Subprocess Management)
Thay vì sử dụng các thư viện Git cồng kềnh, GitSave sử dụng `execFile` từ module `child_process` của Node.js để gọi trực tiếp lệnh `git clone --mirror`.
*   **Tư duy hay:** Sử dụng `--mirror` thay vì `clone` thông thường để sao chép toàn bộ refs, branches và tags của repository.
*   **Timeout Handling:** Có cơ chế kill process nếu lệnh git chạy quá lâu (mặc định 15 giây), tránh treo server khi repo quá lớn hoặc mạng lỗi.

#### b. Bảo mật dữ liệu nhạy cảm (Security Engineering)
*   **Mã hóa Token:** Dự án không lưu Access Token (GitHub/GitLab) dưới dạng văn bản thuần túy. Nó sử dụng `EncryptionService` (AES-256-CBC) để mã hóa token trước khi lưu vào SQLite. Điều này bảo vệ dữ liệu nếu file database bị lộ.
*   **Sanitization:** Có file `sanatize.ts` để lọc các ký tự đặc biệt trong tên repository, tránh lỗi file system hoặc shell injection khi tạo thư mục backup.

#### c. Hệ thống Lưu trữ linh hoạt (SMB & Local)
*   Hỗ trợ backup ra thư mục nội bộ hoặc đẩy thẳng lên **SMB Share** (Samba/Windows Share).
*   Logic xử lý: Nếu dùng SMB, hệ thống sẽ clone về thư mục `/tmp` tạm thời, sau đó dùng `smbclient` để đẩy lên server và xóa file tạm.

#### d. Dynamic Cron Scheduling
Hệ thống không cố định các lịch chạy. Khi server khởi động hoặc khi người dùng cập nhật Schedule, hàm `scheduleCronJobs()` sẽ:
1. Quét database.
2. Dừng các job cũ.
3. Tạo lại các job mới dựa trên biểu thức Cron được sinh ra tự động từ input của người dùng (every X minutes/hours/days).

---

### 4. Tóm tắt luồng hoạt động của Project (System Workflow)

1.  **Khởi tạo (Setup):** Người dùng tạo tài khoản Admin đầu tiên thông qua trang `/setup`. Prisma sẽ tạo file `GitSave.db`.
2.  **Cấu hình (Configuration):** Người dùng thêm `Access Token` (được mã hóa khi lưu) và tạo `Schedule`.
3.  **Lập lịch (Scheduling):** `node-cron` nhận biểu thức cron (ví dụ: `*/30 * * * *`) và đăng ký tác vụ vào bộ nhớ Node.js.
4.  **Thực thi (Execution):**
    *   Đến giờ hẹn, server gọi lệnh `git clone --mirror`.
    *   Nếu thành công, ghi log vào bảng `backupHistory`.
    *   Kiểm tra số lượng bản backup hiện có. Nếu vượt quá số lượng `keepLast` (ví dụ: chỉ giữ 5 bản), nó sẽ tự động xóa bản cũ nhất (`fs.rmSync`).
5.  **Giám sát (Monitoring):** Người dùng theo dõi trạng thái thành công/thất bại qua Dashboard được build bằng Astro + React.

### Đánh giá chung:
Dự án có **tư duy lập trình rất chuyên nghiệp**. Việc sử dụng các công nghệ như `Astro` cho thấy tác giả quan tâm đến trải nghiệm người dùng, còn tầng Backend được cấu trúc theo Services cho thấy kinh nghiệm về xử lý hệ thống bền vững. Đây là một boilerplate tốt để học tập về cách xây dựng công cụ Self-hosted.