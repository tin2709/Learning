Dựa trên mã nguồn và cấu trúc thư mục của kho lưu trữ **Reactive Resume** (phiên bản v4/v5), dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án được xây dựng theo mô hình **Monorepo** (quản lý nhiều dự án trong một kho lưu trữ) sử dụng công cụ **Nx**.

*   **Frontend (Apps):**
    *   **React (Vite):** Framework chính cho giao diện người dùng, đảm bảo tốc độ render nhanh và trải nghiệm mượt mà.
    *   **Tailwind CSS:** Sử dụng để thiết kế giao diện (UI) nhanh chóng, nhất quán.
    *   **Zustand & Zundo:** Quản lý trạng thái (State Management) cực kỳ nhẹ nhàng, hỗ trợ tính năng Undo/Redo (hoàn tác) dữ liệu khi chỉnh sửa resume.
    *   **TanStack Query (React Query):** Quản lý việc gọi API, caching và đồng bộ dữ liệu giữa Client và Server.
    *   **LinguiJS:** Quản lý đa ngôn ngữ (i18n).
*   **Backend (Apps):**
    *   **NestJS:** Framework Node.js mạnh mẽ, viết bằng TypeScript, cung cấp cấu trúc mã nguồn rõ ràng theo module.
    *   **Prisma ORM:** Công cụ giao tiếp với cơ sở dữ liệu (PostgreSQL), giúp quản lý schema và truy vấn dữ liệu an toàn (Type-safe).
*   **Hạ tầng & Lưu trữ:**
    *   **PostgreSQL:** Cơ sở dữ liệu quan hệ chính.
    *   **Minio:** Lưu trữ đối tượng (Object Storage) tương thích với S3, dùng để lưu ảnh đại diện (avatar) và các file PDF resume đã xuất.
    *   **Browserless (Puppeteer):** Chạy trình duyệt Chrome ở chế độ headless để chuyển đổi giao diện HTML/CSS sang file PDF chất lượng cao.
    *   **Docker & Docker Compose:** Đóng gói toàn bộ ứng dụng để dễ dàng triển khai (Self-hosting).

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Reactive Resume rất đặc biệt ở chỗ nó tách biệt hoàn toàn phần "Quản lý" và phần "Hiển thị":

*   **Tách biệt Artboard (`apps/artboard`):** Đây là một ứng dụng React riêng biệt chỉ dùng để render mẫu resume. Khi bạn chỉnh sửa trong Builder, dữ liệu được gửi sang Artboard qua `postMessage`. Điều này giúp cô lập môi trường render, tránh xung đột CSS giữa giao diện người điều khiển và mẫu resume.
*   **Kiến trúc Shared Libs:** Logic quan trọng được chia thành các thư viện dùng chung (`libs/`):
    *   `dto`: Các định dạng dữ liệu trao đổi giữa Client và Server.
    *   `schema`: Định nghĩa cấu trúc dữ liệu của một bản resume (dựa trên chuẩn JSON Resume).
    *   `ui`: Chứa các thành phần giao diện dùng chung (buttons, inputs...).
    *   `utils`: Các hàm bổ trợ xử lý ngày tháng, màu sắc, định dạng file.
*   **Schema-driven:** Mọi mẫu resume đều tuân thủ một bộ quy tắc dữ liệu (Schema) nghiêm ngặt. Điều này cho phép người dùng đổi mẫu (template) ngay lập tức mà không mất dữ liệu.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **HTML-to-PDF Rendering:** Thay vì dùng các thư viện vẽ PDF phức tạp, dự án dùng Puppeteer để "chụp ảnh" trang web Artboard và in ra định dạng PDF. Kỹ thuật này giữ nguyên độ chính xác của CSS và font chữ Google Fonts.
*   **AI Integration:** Tích hợp OpenAI SDK cho phép người dùng cải thiện câu văn, sửa lỗi chính tả hoặc thay đổi giọng văn (tone) của resume ngay trong trình soạn thảo.
*   **Optimistic Updates:** Khi người dùng thay đổi thông tin, giao diện sẽ cập nhật ngay lập tức (thông qua Zustand), sau đó mới gửi dữ liệu lưu xuống server ngầm, tạo cảm giác không có độ trễ.
*   **Thao tác kéo thả (Drag-and-drop):** Sử dụng `@dnd-kit` để cho phép người dùng tùy biến thứ tự các phần (Experience, Education...) một cách linh hoạt.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Luồng hoạt động của Reactive Resume có thể tóm tắt qua các bước sau:

1.  **Khởi tạo:** Người dùng đăng nhập (qua Email hoặc Google/GitHub OAuth). Client tải danh sách resume từ Server (NestJS) thông qua TanStack Query.
2.  **Soạn thảo (Builder):**
    *   Khi người dùng nhập dữ liệu vào `apps/client`, trạng thái (state) được cập nhật trong **Zustand store**.
    *   Store gửi dữ liệu qua một `iframe` chứa ứng dụng `apps/artboard`.
    *   `artboard` nhận dữ liệu, render mẫu resume tương ứng với CSS và font chữ được chọn.
3.  **Lưu trữ:** Dữ liệu được tự động lưu vào **PostgreSQL** thông qua API. Các tài nguyên hình ảnh được tải lên **Minio**.
4.  **Xuất bản & In ấn (Export):**
    *   Người dùng nhấn "Download PDF".
    *   Client gửi yêu cầu đến Server.
    *   Server khởi chạy một instance của **Puppeteer** (qua Browserless).
    *   Puppeteer truy cập vào URL của bản resume đó (trang Preview), chờ tải font xong, sau đó thực hiện lệnh in ra file PDF.
    *   File PDF được lưu tạm vào Minio và trả về link tải cho người dùng.
5.  **Chia sẻ:** Người dùng có thể bật chế độ "Public". Một đường dẫn duy nhất được tạo ra để nhà tuyển dụng có thể xem trực tuyến mà không cần đăng nhập.

**Kết luận:** Dự án là một ví dụ điển hình về việc áp dụng kiến trúc **Monorepo hiện đại**, kết hợp giữa sức mạnh của **NestJS** ở backend và sự linh hoạt của **React** ở frontend, tối ưu hóa cho việc in ấn chất lượng cao thông qua công nghệ trình duyệt không đầu (Headless Browser).