Dựa trên mã nguồn của dự án **myDrive**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động:

---

### 1. Công nghệ Cốt lõi (Core Technologies)

Dự án sử dụng mô hình Fullstack hiện đại với TypeScript là ngôn ngữ chủ đạo:

*   **Backend:**
    *   **Node.js & Express:** Framework chính để xây dựng RESTful API.
    *   **TypeScript:** Cung cấp kiểu dữ liệu chặt chẽ, giúp giảm thiểu lỗi logic.
    *   **MongoDB & Mongoose:** Cơ sở dữ liệu NoSQL để lưu trữ metadata (thông tin file, thư mục, người dùng).
    *   **Lưu trữ (Storage):** Hỗ trợ đa nền tảng thông qua **Filesystem (Local)** hoặc **Amazon S3**.
    *   **Xử lý Media:** `Sharp` (xử lý hình ảnh) và `Fluent-ffmpeg` (trích xuất thumbnail từ video).

*   **Frontend:**
    *   **React (Vite):** Thư viện UI với tốc độ build cực nhanh nhờ Vite.
    *   **Redux Toolkit:** Quản lý trạng thái toàn cục (UI, thông tin upload, lựa chọn file).
    *   **React Query (TanStack Query):** Quản lý server state, caching và tự động fetch lại dữ liệu.
    *   **Tailwind CSS:** Framework CSS để thiết kế giao diện responsive và hiện đại.

*   **Infrastructure & DevOps:**
    *   **Docker & Docker Compose:** Đóng gói ứng dụng và các dịch vụ đi kèm (MongoDB).
    *   **GitHub Actions:** Tự động hóa quá trình build và push Docker image (CI/CD).

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Design Thinking)

Kiến trúc của myDrive được thiết kế theo hướng **Layered Architecture (Kiến trúc phân lớp)** nhằm tách biệt trách nhiệm:

*   **Controller Layer:** Tiếp nhận Request, điều phối các Service và trả về Response (vd: `file-controller.ts`).
*   **Service Layer:** Chứa logic nghiệp vụ quan trọng như mã hóa dữ liệu, tính toán dung lượng (vd: `chunk-service.ts`, `file-service.ts`).
*   **Database Access Layer (DB Utils):** Tách biệt các câu lệnh truy vấn MongoDB khỏi Service layer để dễ dàng bảo trì hoặc thay đổi DB sau này (vd: `fileDB.ts`, `folderDB.ts`).
*   **Middleware:** Xử lý các tác vụ cắt ngang (cross-cutting concerns) như:
    *   **Authentication:** Kiểm tra JWT (Access/Refresh Token).
    *   **Validation:** Sử dụng `express-validator` để kiểm tra tính hợp lệ của dữ liệu đầu vào trước khi vào Controller.
*   **Tư duy mã hóa "At Rest":** Dữ liệu được mã hóa ngay khi lưu trữ, đảm bảo dù quản trị viên server cũng không xem được nội dung file nếu không có key.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

*   **Mã hóa AES-256-CBC:** Đây là điểm sáng nhất. Mỗi người dùng có một khóa mã hóa riêng. File được mã hóa "on-the-fly" (mã hóa theo luồng) khi upload và giải mã khi download, đảm bảo an toàn tối đa.
*   **Xử lý Luồng (Streaming):**
    *   Sử dụng `Busboy` để xử lý việc upload file lớn theo luồng (streaming multipart data), giúp giảm tải bộ nhớ RAM.
    *   Hỗ trợ **Byte-range requests** cho video: Cho phép người dùng tua video ngay cả khi file đang bị mã hóa. Hệ thống sẽ tính toán block dữ liệu và IV (Initialization Vector) tương ứng để giải mã đúng phân đoạn người dùng yêu cầu.
*   **Cơ chế Token Kép (Access & Refresh Tokens):** Tăng cường bảo mật. Access token ngắn hạn được lưu trong HttpOnly Cookie, Refresh token dùng để lấy token mới mà không cần đăng nhập lại.
*   **PWA (Progressive Web App):** Hỗ trợ Service Worker để cache các thumbnail, giúp ứng dụng hoạt động mượt mà như app bản địa trên mobile.
*   **Quản lý Upload phức tạp:** Hỗ trợ upload cả thư mục (giữ nguyên cấu trúc cây) và quản lý tiến trình (progress bar) thông qua Redux.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow)

Dựa trên cấu trúc file, dự án hoạt động theo các luồng chính sau:

#### A. Luồng Đăng ký/Đăng nhập:
1.  Người dùng gửi thông tin qua `user-router`.
2.  `user-service` băm mật khẩu (bcrypt) và **tạo cặp khóa mã hóa riêng** cho người dùng.
3.  JWT được tạo và trả về cho trình duyệt dưới dạng **HttpOnly Cookie**.

#### B. Luồng Upload File:
1.  Người dùng chọn file -> Frontend gọi `uploadFileAPI`.
2.  `file-router` tiếp nhận, `authFullUser` middleware kiểm tra quyền.
3.  `getBusboyData` nhận luồng dữ liệu -> Tạo IV ngẫu nhiên -> Mã hóa AES-256 luồng dữ liệu đó.
4.  Luồng đã mã hóa được đẩy vào S3 hoặc Filesystem.
5.  Metadata (tên file đã mã hóa, IV, owner ID) được lưu vào MongoDB.
6.  Một worker ngầm (`Sharp`/`FFMPEG`) sẽ tạo thumbnail và lưu trữ riêng.

#### C. Luồng Download/Xem file:
1.  Frontend yêu cầu file qua ID.
2.  Backend kiểm tra quyền sở hữu hoặc token chia sẻ công khai.
3.  Lấy khóa mã hóa của người dùng và IV của file từ DB.
4.  Mở luồng đọc từ storage -> Đưa qua bộ giải mã (Decipher) -> Trả luồng dữ liệu thô về cho client.

#### D. Luồng Chia sẻ (Sharing):
1.  Người dùng tạo link chia sẻ (Public hoặc One-time).
2.  Hệ thống tạo một token truy cập tạm thời gắn vào metadata của file.
3.  Người nhận click link -> Backend kiểm tra token hợp lệ -> Cho phép giải mã và tải file mà không cần tài khoản.

---
**Kết luận:** myDrive là một dự án có độ hoàn thiện cao, chú trọng đặc biệt vào **quyền riêng tư (Privacy)** và **hiệu suất xử lý file lớn**. Việc kết hợp giữa TypeScript, mã hóa luồng và kiến trúc Service-oriented khiến nó trở thành một hệ thống lưu trữ đám mây cá nhân rất mạnh mẽ.