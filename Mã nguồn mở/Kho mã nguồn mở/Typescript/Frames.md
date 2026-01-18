Dưới đây là bản phân tích chi tiết và tóm tắt dự án **Frames** dưới dạng file README tiếng Việt, tập trung vào công nghệ, kiến trúc và luồng hoạt động của hệ thống.

---

# 🚀 Frames: Nền Tảng Streaming Media Hiện Đại

**Frames** là một dịch vụ SVOD (Subscription Video on Demand) mã nguồn mở, cho phép người dùng tự xây dựng "Netflix cá nhân". Dự án được chuyển đổi từ việc chỉ hỗ trợ Google Drive sang hỗ trợ đa nền tảng lưu trữ (S3, Dropbox, Local, Google Drive).

## 🛠 1. Công Nghệ Cốt Lõi (Core Technologies)

### **Backend (Hệ thống máy chủ)**
*   **NestJS (Node.js Framework):** Sử dụng kiến trúc Modular để quản lý mã nguồn linh hoạt và dễ mở rộng.
*   **Prisma ORM:** Công cụ truy vấn cơ sở dữ liệu mạnh mẽ, hỗ trợ Type-safety.
*   **PostgreSQL & pgvector:** Lưu trữ dữ liệu quan hệ và vector hóa dữ liệu (embedding) để phục vụ tìm kiếm gợi ý bằng AI.
*   **Redis & BullMQ:** Quản lý hàng đợi (queue) cho các tác vụ nặng như quét thư viện media (scanning) và xử lý ngầm.
*   **Socket.io (PondSocket):** Xử lý giao tiếp thời gian thực cho tính năng GroupWatch (xem chung) và thông báo.

### **Frontend (Giao diện người dùng)**
*   **React & Vite:** Framework giao diện nhanh, mượt mà.
*   **TanStack Router:** Quản lý routing phức tạp trong ứng dụng.
*   **Tailwind CSS:** Framework CSS cho giao diện Dark-mode hiện đại.
*   **Vime/Video.js:** Tùy biến trình phát video cao cấp hỗ trợ Subtitles, AirPlay, và Transcoding.

### **Infrastructure (Hạ tầng & Deployment)**
*   **Docker & Docker Compose:** Đóng gói ứng dụng thành các container.
*   **Docker Buildx:** Hỗ trợ build đa kiến trúc (x86 và ARM/Apple Silicon).
*   **FFmpeg:** Xử lý video, tạo ảnh thumbnail và chuyển mã (transcoding).

---

## 🏗 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án được xây dựng theo hướng **Modular Monolith**, nơi mỗi tính năng (Auth, Media, Scanner, Playback) là một module độc lập trong NestJS nhưng chạy chung một tiến trình.

*   **Lớp trừu tượng lưu trữ (Storage Abstraction):** Hệ thống không phụ thuộc vào một nhà cung cấp cụ thể. Nhờ thư viện `@eleven-am/nestjs-storage`, Frames có thể đọc file từ Local, S3, hay Dropbox thông qua một giao diện lập trình duy nhất.
*   **Kiến trúc dựa trên phân quyền (Resource-based Authorization):** Sử dụng `@eleven-am/authorizer` để kiểm soát quyền truy cập chi tiết (ví dụ: chỉ Admin mới được sửa metadata, người dùng chỉ được xem media trong nhóm của họ).
*   **Hybrid Database:** Kết hợp giữa dữ liệu quan hệ (Postgres) và tìm kiếm ngữ nghĩa (Semantic Search) thông qua OpenAI Embeddings.

---

## 🔑 3. Các Kỹ Thuật Chính (Key Techniques)

### **A. Hệ thống Quét Media Thông Minh (Scanner Service)**
Hệ thống không chỉ đọc file mà còn:
1.  Nhận diện cấu trúc thư mục (Phim lẻ vs Phim bộ).
2.  Kết nối với **TMDB API**, **Fanart.tv**, và **Apple Artwork** để lấy thông tin (poster, diễn viên, nội dung).
3.  Sử dụng **OpenAI API** để tạo vector từ mô tả phim, lưu vào `pgvector` để gợi ý "Phim tương tự" với độ chính xác cao.

### **B. Bảo mật & Xác thực Hiện đại**
*   **WebAuthn (Passkeys):** Hỗ trợ đăng nhập không mật khẩu bằng vân tay hoặc FaceID (sinh trắc học).
*   **OAuth2:** Tích hợp đăng nhập qua các bên thứ ba.
*   **Auth Keys:** Hệ thống khóa mời (Invite keys) để kiểm soát việc đăng ký thành viên mới.

### **C. Luồng Phát Video (Playback & Streaming)**
*   **Chuyển mã Alpha (Transcoding):** Tự động chuyển đổi MP4 sang HLS (HTTP Live Streaming) để tối ưu hóa băng thông và hỗ trợ nhiều độ phân giải.
*   **Subtitles Expanded:** Kỹ thuật tách và xử lý file phụ đề (VTT/SRT) cho tất cả các ngôn ngữ, hỗ trợ đồng bộ hóa thời gian thực.

---

## 🔄 4. Tóm Tắt Luồng Hoạt Động (Workflow Summary)

1.  **Thiết lập (Setup):** Người dùng cài đặt qua Docker, cung cấp API Key (TMDB, OpenAI). Hệ thống khởi tạo DB thông qua Prisma Migrations (sử dụng `start.sh` để kiểm tra kết nối DB trước khi chạy).
2.  **Kết nối Thư viện (Library Connection):** Admin thêm đường dẫn thư mục (Local hoặc Cloud).
3.  **Quét & Chỉ mục (Scan & Index):** 
    *   Hàng đợi BullMQ bắt đầu quét file. 
    *   Xác định định danh phim (Identification).
    *   Tải metadata và ảnh về cache. 
    *   Tạo vector embedding cho AI.
4.  **Khám phá (Discovery):** Người dùng truy cập giao diện, hệ thống hiển thị phim theo xu hướng, thể loại hoặc gợi ý cá nhân hóa từ AI.
5.  **Phát & Tương tác (Stream & Interact):** 
    *   Người dùng nhấn Play, Backend cấp quyền (Authorization). 
    *   Trình phát video gọi stream từ Storage. 
    *   Người dùng có thể tạo phòng **GroupWatch**, hệ thống sử dụng WebSockets để đồng bộ thời gian phát giữa các thành viên.
6.  **Quản lý (Admin):** Admin theo dõi sức khỏe hệ thống qua module **Health**, quản lý các bộ sưu tập (Picks) và phân quyền người dùng.

---

## 🏁 Kết luận
**Frames** không chỉ là một trình phát video, mà là một hệ thống quản trị nội dung media phức tạp, áp dụng những kỹ thuật lập trình tiên tiến nhất như AI Vector Search, xác thực WebAuthn và kiến trúc module hóa cao độ. Project này là lựa chọn hoàn hảo cho những ai muốn tự chủ hoàn toàn kho dữ liệu phim ảnh của mình.