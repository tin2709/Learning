Dưới đây là bản phân tích chi tiết về dự án **Photoview** dựa trên mã nguồn và tài liệu bạn cung cấp:

### 1. Phân tích Công nghệ cốt lõi (Core Tech Stack)

Dự án được xây dựng theo mô hình Client-Server hiện đại, tập trung vào hiệu suất xử lý media:

*   **Backend (API):**
    *   **Ngôn ngữ:** Go (Golang) - Lựa chọn tối ưu cho việc xử lý song song và tương tác hệ thống.
    *   **Framework GraphQL:** `gqlgen` - Tự động tạo code từ schema, đảm bảo type-safety chặt chẽ giữa frontend và backend.
    *   **ORM:** `GORM` - Hỗ trợ đa cơ sở dữ liệu (SQLite, MySQL, PostgreSQL).
    *   **Xử lý Media:** `FFmpeg` (Video), `ExifTool` (Metadata), `ImageMagick` (Image processing), `libheif` (Hỗ trợ định dạng ảnh Apple).
    *   **Nhận diện khuôn mặt:** `go-face` (dựa trên thư viện C++ `dlib`).
*   **Frontend (UI):**
    *   **Ngôn ngữ/Framework:** TypeScript & React.
    *   **Build Tool:** Vite (thay thế cho CRA để có tốc độ build cực nhanh).
    *   **GraphQL Client:** Apollo Client.
    *   **Styling:** Tailwind CSS.
*   **Infrastructure:**
    *   **Containerization:** Docker & Docker Compose.
    *   **CI/CD:** GitHub Actions (tự động build image và kiểm thử).

### 2. Tư duy Kiến trúc (Architecture Insights)

Photoview sử dụng kiến trúc **Filesystem-driven (Dựa trên hệ thống tệp)**:

*   **Mapping 1:1:** Không giống như Google Photos trộn lẫn mọi thứ, Photoview ánh xạ trực tiếp cấu trúc thư mục trên ổ cứng thành các Album trên web. Điều này giúp người dùng quản lý ảnh bằng File Explorer truyền thống vẫn hiệu quả.
*   **GraphQL API First:** Toàn bộ giao tiếp giữa UI và Server đều qua GraphQL. Kiến trúc này cho phép UI chỉ tải đúng những gì cần thiết (ví dụ: chỉ lấy `thumbnail_url` mà không lấy toàn bộ metadata khi xem danh sách).
*   **N+1 Optimization:** Sử dụng **Dataloaders** (`api/dataloader/`) để gom các yêu cầu truy vấn dữ liệu (như lấy URL ảnh cho danh sách 100 ảnh) thành một câu lệnh SQL duy nhất, tránh làm treo database.
*   **Worker Pool Pattern:** Quá trình quét ảnh (Scanner) và mã hóa video (Encoding) được chạy dưới dạng các background workers để không làm tắc nghẽn API chính.

### 3. Các kỹ thuật chính (Key Technical Implementations)

*   **Xử lý Metadata (EXIF):** Có logic phức tạp để xử lý múi giờ (`api/graphql/models/media_exif.go`). Nó lưu trữ cả thời gian chụp và độ lệch múi giờ (offset) để hiển thị chính xác timeline dù ảnh được chụp ở các quốc gia khác nhau.
*   **Nhận diện khuôn mặt (Face Recognition):** Tích hợp sâu với thư viện native (`dlib`). Quy trình bao gồm: Phát hiện khuôn mặt -> Trích xuất vector đặc trưng (128 chiều) -> Lưu vào DB -> So sánh vector để gom nhóm người.
*   **Security (Bảo mật):**
    *   Sử dụng `bcrypt` để băm mật khẩu.
    *   Xác thực qua `Cookie-token` (HttpOnly) để chống tấn công XSS.
    *   Cơ chế `Share Token` cho phép chia sẻ Album/Ảnh bằng link công khai kèm mật khẩu bảo vệ tùy chọn.
*   **Hardware Acceleration:** Hỗ trợ tăng tốc phần cứng (Intel QSV, NVIDIA NVENC) khi convert video, giúp giảm tải CPU cho server (thường là các máy NAS yếu).
*   **Database Migrations:** Tự quản lý phiên bản database thông qua GORM và các script migration thủ công để xử lý các thay đổi dữ liệu lớn (như chuyển đổi kiểu dữ liệu tọa độ GPS).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Cài đặt:** Người dùng ánh xạ thư mục ảnh (ví dụ: `/home/user/photos`) vào Container Docker tại đường dẫn `/photos`.
2.  **Scanner (Quét):**
    *   Hệ thống duyệt cây thư mục.
    *   So sánh `PathHash` để biết ảnh nào mới, ảnh nào đã xóa.
    *   Trích xuất Metadata (ngày chụp, tọa độ, thiết bị).
    *   Tạo ảnh Thumbnail (nhỏ) và High-res (vừa) lưu vào bộ nhớ đệm (`media_cache`).
3.  **Indexing:** Dữ liệu được lưu vào Database. Lúc này giao diện web mới bắt đầu hiển thị ảnh.
4.  **Phục vụ:**
    *   Khi người dùng mở web, UI gửi query GraphQL.
    *   Server trả về các URL thumbnail.
    *   Ảnh gốc chỉ được truy cập khi người dùng nhấn "Download" hoặc xem toàn màn hình (để tiết kiệm băng thông).
5.  **Nhận diện:** Background task tự động phân tích khuôn mặt và gợi ý gom nhóm các thành viên trong gia đình.

### Đánh giá chung
Đây là một dự án có **kiến trúc rất sạch (clean architecture)**. Việc tách biệt logic xử lý ảnh (Scanner) và logic hiển thị (GraphQL API) giúp hệ thống ổn định. Tuy nhiên, do phụ thuộc vào nhiều thư viện native (CGO) như `dlib` và `imagemagick`, việc tự build thủ công (không dùng Docker) sẽ khá phức tạp về mặt dependency.