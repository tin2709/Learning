Dưới đây là bản phân tích chi tiết và tóm tắt dự án **Flame** dựa trên mã nguồn bạn đã cung cấp, được trình bày dưới dạng một file README chuyên sâu bằng tiếng Việt.

---

# Phân Tích Hệ Thống Startpage Flame

Flame là một ứng dụng startpage tự lưu trữ (self-hosted) mạnh mẽ, giúp người dùng quản lý các ứng dụng và dấu trang (bookmarks) thông qua giao diện web trực quan.

## 1. Công Nghệ Cốt Lõi (Tech Stack)

### Backend (Node.js & Express)
*   **Framework:** Express.js - Xử lý API RESTful và phục vụ file tĩnh.
*   **Database:** SQLite - Lựa chọn tối ưu cho ứng dụng self-hosted nhờ tính gọn nhẹ, không cần cấu hình server DB riêng.
*   **ORM:** Sequelize - Quản lý mô hình dữ liệu (Models) và quan hệ giữa chúng.
*   **Migration:** Umzug - Xử lý việc nâng cấp cấu trúc database tự động khi cập nhật phiên bản mới.
*   **Authentication:** JSON Web Token (JWT) - Xác thực người dùng thông qua mật khẩu được cấu hình qua biến môi trường.
*   **Real-time:** WebSocket (`ws`) - Cập nhật dữ liệu thời tiết trực tiếp đến giao diện người dùng mà không cần tải lại trang.

### Frontend (React & TypeScript)
*   **UI Library:** React.js (v17+) với TypeScript - Đảm bảo tính chặt chẽ về kiểu dữ liệu.
*   **State Management:** Redux với Redux-Thunk - Quản lý trạng thái toàn cục (config, apps, bookmarks, themes).
*   **Styling:** CSS Modules - Tránh xung đột CSS giữa các component.
*   **Drag & Drop:** `react-beautiful-dnd` - Cho phép người dùng kéo thả để sắp xếp thứ tự ứng dụng/bookmarks.
*   **Icons:** Material Design Icons (MDI) & Skycons (cho thời tiết).

### Deployment & DevOps
*   **Containerization:** Docker & Docker Compose - Hỗ trợ đa kiến trúc (amd64, arm64, armv7).
*   **Orchestration:** Kubernetes (K8s) - Tích hợp sẵn manifests và Ingress API để tự động lấy thông tin ứng dụng.
*   **CI/CD:** Skaffold - Tối ưu hóa quy trình phát triển trong môi trường Kubernetes.

---

## 2. Tư Duy Kiến Trúc (Architectural Thinking)

### Cấu trúc "Config-Driven"
Flame được thiết kế để **không cần chỉnh sửa file thủ công**. Mọi cài đặt từ giao diện, màu sắc (Theme) đến danh sách ứng dụng đều được quản lý qua GUI và lưu vào `config.json` hoặc SQLite. 

### Thiết kế Stateless & Persistent Data
Hệ thống chia tách rõ rệt giữa mã nguồn và dữ liệu:
*   Mọi dữ liệu động nằm trong thư mục `/data` (DB, Uploads, Config).
*   Điều này giúp việc sao lưu và cập nhật phiên bản Docker cực kỳ đơn giản (chỉ cần mount volume `/app/data`).

### Khả năng Mở rộng (Extensibility)
*   **Custom Search Providers:** Cho phép người dùng tự định nghĩa quy tắc tìm kiếm qua các tiền tố (ví dụ `/g` cho Google).
*   **Custom CSS:** Cung cấp trình biên tập CSS ngay trên trình duyệt để thay đổi hoàn toàn giao diện mà không cần build lại code.

---

## 3. Các Kỹ Thuật Chính (Key Techniques)

### Tích hợp Docker & Kubernetes API
Đây là tính năng thông minh nhất của Flame:
*   **Docker:** Sử dụng `axios` gọi đến Docker Socket (`/var/run/docker.sock`) để đọc các `Labels` của container. Nếu container có label `flame.type=app`, nó sẽ tự động xuất hiện trên startpage.
*   **Kubernetes:** Sử dụng `@kubernetes/client-node` để quét các Ingress trong cluster và tự động tạo lối tắt truy cập dựa trên Annotations.

### Hệ thống Theme linh hoạt
Flame sử dụng **CSS Variables** (`--color-primary`, `--color-accent`,...). Khi người dùng chọn một Theme hoặc tự tạo màu mới, Redux sẽ cập nhật các biến này vào `document.body.style`, giúp giao diện thay đổi tức thì mà không cần load lại CSS file.

### Xử lý Migration Dữ liệu
Sử dụng Umzug để quản lý các bước thay đổi DB. Ví dụ: Khi nâng cấp từ phiên bản cũ lên phiên bản hỗ trợ "App Description", script migration sẽ tự động thêm cột vào SQLite mà không làm mất dữ liệu cũ của người dùng.

### Bảo mật phân tầng
*   **Public/Private Visibility:** Mỗi ứng dụng hoặc danh mục bookmark có thuộc tính `isPublic`. Nếu người dùng chưa đăng nhập, họ chỉ thấy các mục Public. Các mục nhạy cảm và phần Cài đặt sẽ bị khóa bởi JWT Middleware.

---

## 4. Luồng Hoạt Động (Workflow Summary)

### Khởi động Hệ thống (Bootstrapping)
1.  **Server Start:** `server.js` chạy -> `initApp()` kiểm tra sự tồn tại của các thư mục và file cấu hình mặc định trong `/data`.
2.  **Database:** `connectDB()` thực thi các bản vá (migrations) chưa chạy thông qua Umzug.
3.  **Cron Jobs:** Hệ thống khởi tạo các tác vụ chạy ngầm:
    *   Cập nhật thời tiết mỗi 15 phút.
    *   Dọn dẹp dữ liệu thời tiết cũ sau mỗi 4 giờ.
4.  **Socket:** Mở kết nối WebSocket để sẵn sàng đẩy dữ liệu thời tiết xuống Client.

### Luồng Xử lý Yêu cầu (Request Flow)
1.  **Client:** Truy cập trang web -> React App được tải.
2.  **Initialization:** `App.tsx` gọi API `GET /api/config` để lấy cấu hình và Theme.
3.  **Auth:** Nếu có token trong `localStorage`, hệ thống tự động kiểm tra tính hợp lệ qua `POST /api/auth/validate`.
4.  **Data Fetching:** Home page gọi đồng thời các API lấy danh sách Apps và Bookmarks. Middleware `auth.js` sẽ kiểm tra xem có token không để quyết định trả về toàn bộ dữ liệu hay chỉ các mục Public.

### Luồng Tìm kiếm (Search Flow)
1.  Người dùng nhập truy vấn vào `SearchBar`.
2.  `searchParser.ts` phân tích cú pháp:
    *   Nếu bắt đầu bằng `/prefix` (ví dụ `/yt`): Chuyển hướng đến Search Engine tương ứng (YouTube).
    *   Nếu là URL/IP hợp lệ: Chuyển hướng trực tiếp đến địa chỉ đó.
    *   Nếu không có tiền tố: Thực hiện tìm kiếm cục bộ (Local Filter) trong danh sách Apps/Bookmarks trước, nếu không thấy mới chuyển hướng sang Search Engine mặc định.

---

## 5. Tổng Kết
Dự án Flame là một minh chứng điển hình cho việc kết hợp giữa **Node.js (Backend)** và **React (Frontend)** để tạo ra một công cụ tiện ích mạnh mẽ. Với tư duy ưu tiên Docker và tính tùy biến cao, nó giải quyết triệt để bài toán quản lý tài nguyên cho các quản trị viên server cá nhân (Home-lab).