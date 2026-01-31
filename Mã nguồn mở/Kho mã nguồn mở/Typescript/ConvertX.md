Dưới đây là bản phân tích chi tiết về dự án **ConvertX** dựa trên mã nguồn bạn đã cung cấp. ConvertX là một công cụ chuyển đổi tệp tin trực tuyến tự lưu trữ (self-hosted), được thiết kế với tiêu chí tốc độ, nhẹ và hỗ trợ đa dạng định dạng.

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án sử dụng một "stack" hiện đại, tối ưu cho hiệu suất và trải nghiệm nhà phát triển:

*   **Runtime:** **Bun** - Thay thế Node.js để đạt tốc độ thực thi nhanh hơn, tích hợp sẵn trình quản lý gói và trình chạy kiểm thử (test runner).
*   **Web Framework:** **ElysiaJS** - Một framework web cực nhanh cho Bun, hỗ trợ TypeScript hoàn hảo và có cú pháp tương tự Express/Fastify nhưng hiệu suất cao hơn.
*   **Database:** **SQLite (thông qua `bun:sqlite`)** - Lưu trữ dữ liệu người dùng, lịch sử công việc (jobs) và trạng thái chuyển đổi mà không cần cài đặt máy chủ database phức tạp.
*   **Frontend & Rendering:**
    *   **JSX/TSX (Server-side):** Sử dụng `@kitajs/html` để render HTML trực tiếp từ phía server, giúp ứng dụng phản hồi nhanh.
    *   **Tailwind CSS 4.0:** Xử lý giao diện (UI) với hiệu năng cao và tùy biến linh hoạt.
    *   **Vanilla JS:** Sử dụng JavaScript thuần ở phía client (`script.js`, `results.js`) để xử lý upload và polling trạng thái mà không cần các framework nặng như React/Vue.
*   **Conversion Engines:** Đây là "trái tim" của hệ thống, dựa trên các công cụ dòng lệnh (CLI) mã nguồn mở nổi tiếng:
    *   **FFmpeg:** Video/Audio.
    *   **Pandoc/LibreOffice:** Tài liệu văn phòng.
    *   **ImageMagick/GraphicsMagick/Vips:** Hình ảnh.
    *   **Inkscape/resvg:** Vector.
    *   **Assimp:** Mô hình 3D.
    *   **Calibre:** E-book.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của ConvertX đi theo hướng **Modular Monolith** (Khối thống nhất nhưng phân mô-đun):

*   **Mô hình Adapter (Adapter Pattern):** Mỗi công cụ chuyển đổi (ví dụ: FFmpeg, Pandoc) được đóng gói trong một tệp riêng trong `src/converters/`. Tất cả đều tuân thủ một giao diện chung: nhận đường dẫn tệp vào, định dạng đích và trả về kết quả. Điều này giúp việc thêm một công cụ mới cực kỳ dễ dàng mà không ảnh hưởng đến logic cốt lõi.
*   **Server-Driven UI:** Phần lớn logic hiển thị được quyết định bởi Server thông qua JSX. Client chỉ đóng vai trò gửi yêu cầu và cập nhật các đoạn HTML nhận được (gần giống tư tưởng của HTMX).
*   **Xử lý bất đồng bộ (Background Processing):** Việc chuyển đổi tệp không làm nghẽn luồng xử lý của web server. Khi người dùng nhấn "Convert", server sẽ tạo một tiến trình con (child process) và trả về trang kết quả ngay lập tức. Trang này sau đó sẽ tự động cập nhật trạng thái thông qua polling.
*   **Cấu hình qua biến môi trường (Environment-Driven):** Mọi thiết lập từ JWT, Webroot đến các tham số tùy chỉnh cho FFmpeg đều được cấu hình qua `.env`, giúp việc triển khai qua Docker trở nên cực kỳ linh hoạt.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Quản lý Tiến trình Con (`execFile`):** Thay vì dùng `exec` (dễ bị tấn công shell injection), dự án dùng `execFile` từ node:child_process để thực thi các lệnh CLI một cách an toàn.
*   **Xử lý Tệp tin lớn:** Cấu hình `maxRequestBodySize` lên mức tối đa (`Number.MAX_SAFE_INTEGER`) để hỗ trợ người dùng tải lên các tệp video nặng.
*   **Hệ thống Dọn dẹp Tự động:** Một hàm `clearJobs` chạy định kỳ (mặc định 24h) để xóa các tệp đã chuyển đổi và tệp tạm, tránh làm đầy ổ cứng server.
*   **Bảo mật:**
    *   **JWT (JSON Web Token):** Quản lý phiên đăng nhập của người dùng.
    *   **Sanitize Filename:** Sử dụng thư viện `sanitize-filename` để ngăn chặn các cuộc tấn công ghi đè tệp hệ thống bằng cách đổi tên tệp đầu vào.
    *   **Password Hashing:** Sử dụng `Bun.password` để băm mật khẩu an toàn.
*   **Docker Multi-stage Build:** File Dockerfile được chia làm nhiều giai đoạn (install -> prerelease -> release) để giảm kích thước ảnh cuối cùng và tách biệt môi trường phát triển/thực thi.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Khởi tạo:** Khi người dùng truy cập, hệ thống kiểm tra nếu là lần đầu chạy sẽ yêu cầu thiết lập tài khoản admin.
2.  **Tải tệp (Upload):**
    *   Người dùng kéo thả tệp. JavaScript phía client (`script.js`) gửi tệp đến endpoint `/upload`.
    *   Server lưu tệp vào `data/uploads/{userId}/{jobId}/`.
3.  **Lựa chọn định dạng:**
    *   Client gửi định dạng tệp gốc lên server qua `/conversions`.
    *   Server tra cứu trong danh sách `possibleTargets` (tổng hợp từ tất cả các converters) để trả về các định dạng có thể chuyển đổi tương ứng.
4.  **Chuyển đổi (Conversion):**
    *   Khi người dùng nhấn "Convert", một yêu cầu POST gửi đến `/convert`.
    *   Server xác định `converter` phù hợp nhất (ví dụ: dùng `inkscape` cho EMF thay vì `imagemagick`).
    *   Hàm `handleConvert` được gọi bất đồng bộ. Nó sử dụng `execFile` để chạy lệnh CLI của công cụ tương ứng.
    *   Kết quả chuyển đổi lưu vào `data/output/{userId}/{jobId}/`.
5.  **Theo dõi & Tải về:**
    *   Người dùng được chuyển đến trang `/results/{jobId}`.
    *   File `results.js` ở client thực hiện "polling" (gửi yêu cầu liên tục mỗi giây) đến `/progress/{jobId}` để cập nhật thanh tiến trình.
    *   Khi hoàn tất, người dùng có thể tải từng tệp hoặc tải file nén `.tar` của toàn bộ công việc.

### Tổng kết
ConvertX là một ví dụ điển hình về việc tận dụng sức mạnh của các công cụ CLI lâu đời kết hợp với các công nghệ Web hiện đại (Bun, Elysia) để tạo ra một ứng dụng mạnh mẽ, dễ bảo trì và hiệu quả cao.