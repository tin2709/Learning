Dưới đây là phân tích chi tiết về dự án **Fireshare** dựa trên mã nguồn và tài liệu bạn cung cấp:

### 1. Công nghệ Cốt lõi (Core Technologies)
Dự án được xây dựng trên một "stack" hiện đại, tối ưu cho việc xử lý đa phương tiện và tự triển khai (self-hosting):
*   **Frontend:** ReactJS kết hợp với Material UI (MUI). Sử dụng Video.js để làm trình phát video chuyên dụng.
*   **Backend:** Python (Flask framework dựa trên cấu trúc thư mục `app/server`).
*   **Cơ sở dữ liệu:** Sử dụng SQLAlchemy và Alembic (thư mục `migrations`) để quản lý cấu trúc database.
*   **Xử lý Video:** FFmpeg là trái tim của hệ thống, hỗ trợ các codec tiên tiến như H.264, VP9 và đặc biệt là **AV1**.
*   **Hạ tầng/Triển khai:** Docker & Docker Compose. Hỗ trợ tăng tốc phần cứng qua NVIDIA NVENC (GPU).
*   **Web Server/Proxy:** Nginx được tích hợp sẵn để điều phối request và phục vụ file tĩnh.

### 2. Kỹ thuật và Tư duy Kiến trúc
Kiến trúc của Fireshare tập trung vào tính **đơn giản cho người dùng** nhưng **mạnh mẽ ở hậu trường**:
*   **Kiến trúc Phân lớp:** Tách biệt rõ ràng giữa `client` (React) và `server` (Python API).
*   **Tư duy "File-system as Database":** Thay vì bắt người dùng upload thủ công từng file, Fireshare quét (scan) trực tiếp thư mục `/videos`. Cấu trúc thư mục của người dùng chính là các "Folder" phân loại trên giao diện.
*   **Quản lý trạng thái bằng Migration:** Sử dụng Alembic để theo dõi các thay đổi của database (như thêm cột transcoding, hỗ trợ LDAP), giúp việc nâng cấp phiên bản không làm mất dữ liệu.
*   **Containerization (Docker):** Đóng gói mọi phụ thuộc (dependency) như thư viện FFmpeg hay driver NVIDIA vào trong Docker image, giúp việc cài đặt chỉ mất một câu lệnh.

### 3. Các kỹ thuật chính nổi bật
*   **GPU Transcoding (NVENC):** Đây là kỹ thuật khó nhất trong dự án. Fireshare hỗ trợ chuyển mã video bằng GPU NVIDIA để giảm tải cho CPU (nhanh hơn 5-10 lần).
*   **Chuỗi fallback thông minh (Transcoding Fallback Chain):** Nếu GPU không hỗ trợ AV1, nó tự chuyển sang H.264 GPU; nếu không có GPU, nó tự chuyển sang CPU (libaom-av1 hoặc libx264). Điều này đảm bảo video luôn xem được trên mọi thiết bị.
*   **Open Graph Support:** Tự động tạo metadata để khi gửi link qua Discord, Telegram hay SMS, nó sẽ hiển thị hình ảnh xem trước (thumbnail) và tiêu đề video.
*   **LDAP Authentication:** Hỗ trợ kết nối với các hệ thống quản lý người dùng tập trung cho các tổ chức hoặc gia đình có nhiều thành viên.
*   **Background Tasks:** Hệ thống có các tiến trình chạy ngầm (scheduled scan) để tự động cập nhật video mới mà không cần người dùng can thiệp.

### 4. Tóm tắt luồng hoạt động (Workflow)
Luồng xử lý của Fireshare diễn ra như sau:
1.  **Quét (Scanning):** Mỗi 5 phút (mặc định), hệ thống quét thư mục `/videos` để tìm file mới (.mp4, .webm, .mov).
2.  **Trích xuất Metadata:** Hệ thống đọc thông tin video, tạo ảnh đại diện (poster) và lưu vào thư mục `/processed`.
3.  **Chuyển mã (Transcoding - Nếu bật):** Nếu video gốc quá lớn hoặc định dạng khó xem, Fireshare tạo ra các bản sao 720p/1080p bằng codec AV1/H.264 để tối ưu băng thông khi xem từ xa.
4.  **Quản lý (Dashboard):** Admin đăng nhập để chỉnh sửa tiêu đề, mô tả hoặc đặt video ở chế độ riêng tư/công khai.
5.  **Chia sẻ (Sharing):** Hệ thống tạo ra một liên kết (link) duy nhất. Khi người nhận click vào, họ có thể xem video trực tiếp trên trình duyệt mà không cần cài đặt gì thêm.
6.  **Phát trực tuyến (Streaming):** Nginx phục vụ file video theo dạng stream, cho phép người dùng tua nhanh (seek) đến bất kỳ thời điểm nào của video.

**Kết luận:** Fireshare là một giải pháp hoàn hảo cho những ai muốn tự quản lý kho video clip cá nhân (như clip chơi game) mà không muốn phụ thuộc vào các nền tảng lớn như YouTube, đồng thời vẫn giữ được trải nghiệm xem mượt mà và tính năng chia sẻ chuyên nghiệp.