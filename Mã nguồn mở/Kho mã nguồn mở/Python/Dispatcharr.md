Dựa trên mã nguồn và cấu trúc thư mục của dự án **Dispatcharr**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của hệ thống quản lý IPTV này:

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dispatcharr là một ứng dụng Full-stack hiện đại, kết hợp giữa sức mạnh xử lý media của Python và tính tương tác của React:

*   **Backend Framework:** **Django 5.2 (Python 3.13)**. Đây là phiên bản rất mới, sử dụng **Django REST Framework (DRF)** để cung cấp API cho Frontend và các ứng dụng bên thứ ba (Plex, Jellyfin).
*   **Xử lý bất đồng bộ:** **Celery** kết hợp với **Redis**. Hệ thống dựa cực kỳ nhiều vào các tác vụ nền (background tasks) để refresh M3U, tải dữ liệu EPG, và thực hiện ghi chương trình (DVR).
*   **Quản lý luồng Media:** Sử dụng các engine mạnh mẽ như **FFmpeg, Streamlink, VLC, và yt-dlp**. Dự án có khả năng chuyển mã (transcode) và proxy luồng trực tiếp.
*   **Real-time Communication:** **Django Channels** và **Redis PubSub**. Dùng để cập nhật trạng thái viewer, tiến độ tải EPG lên giao diện người dùng ngay lập tức qua WebSockets.
*   **Học máy (ML):** Tích hợp **PyTorch (CPU-only)** và **Sentence-Transformers**. Đây là điểm đặc biệt, dùng để so khớp (matching) tên kênh và dữ liệu EPG một cách thông minh bằng thuật toán ngữ nghĩa thay vì chỉ so khớp chuỗi thông thường.
*   **Frontend:** **React** với **Vite**, sử dụng bộ UI **Mantine**. Trạng thái được quản lý bởi **Zustand** (qua các file `store/*.jsx`).
*   **Package Management:** Sử dụng **UV** - một công cụ quản lý thư viện Python siêu nhanh mới nổi, thay thế cho pip/poetry.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dispatcharr được thiết kế theo mô hình **Modular Monolith** (Khối đồng nhất nhưng mô-đun hóa cao):

*   **Chia nhỏ theo nghiệp vụ (Apps):** Mỗi chức năng lớn là một Django App riêng biệt:
    *   `apps/channels`: Quản lý danh sách kênh và logic gộp kênh.
    *   `apps/m3u`: Xử lý việc nhập và lọc playlist từ các nhà cung cấp.
    *   `apps/epg`: Xử lý lịch phát sóng.
    *   `apps/proxy`: Đây là "trái tim" về kỹ thuật, xử lý việc truyền dẫn dữ liệu video (TS và HLS Proxy).
*   **Kiến trúc hướng sự kiện (Event-driven):** Sử dụng **Django Signals** (`apps/channels/signals.py`) một cách triệt để. Ví dụ: khi một bản ghi "Recording" được lưu, một tín hiệu sẽ tự động tính toán thời gian và lập lịch tác vụ Celery để bắt đầu ghi hình.
*   **Khả năng mở rộng qua Plugin:** Hệ thống có một `loader.py` trong `apps/plugins` cho phép người dùng viết thêm code Python để mở rộng tính năng mà không cần sửa mã nguồn gốc.
*   **Thiết kế Cloud-native:** Hỗ trợ Docker AIO (All-in-one) cho người dùng phổ thông và Modular Deployment (tách rời DB, Redis, Worker) cho hệ thống lớn.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Stream Proxying & Relay:** Dispatcharr không chỉ chuyển hướng URL. Nó tạo ra một proxy trung gian. Khi một client (như VLC) kết nối, Dispatcharr mở một luồng tới server IPTV, nhận dữ liệu, xử lý (nếu cần) và truyền tiếp cho client. Kỹ thuật này giúp ẩn IP gốc và quản lý số lượng kết nối tối đa.
*   **Nhận dạng kênh thông minh (Stable Identity):** Sử dụng `stream_hash` (SHA256) dựa trên các thuộc tính như tên kênh, URL hoặc ID nhà cung cấp. Điều này giúp hệ thống nhận diện đúng kênh ngay cả khi nhà cung cấp thay đổi mật khẩu hoặc cấu trúc URL.
*   **Quản lý bộ nhớ đệm (Caching):** Logo kênh và dữ liệu EPG được cache tại `/data` để giảm tải cho server gốc và tăng tốc độ tải trang Guide.
*   **HDHomeRun Emulation:** Giả lập giao thức của thiết bị phần cứng HDHomeRun để các phần mềm như Plex/Emby có thể tự động nhận diện Dispatcharr như một bộ thu truyền hình kỹ thuật số.
*   **Bảo mật phân tầng:** Kết hợp JWT (cho UI) và API Key (cho các thiết bị stream), đồng thời có hệ thống `network_access_allowed` để giới hạn truy cập theo dải IP.

### 4. Luồng hoạt động chính (Operational Flow)

1.  **Nhập dữ liệu (Ingestion):** Người dùng thêm M3U Account -> Celery Task được kích hoạt -> Tải playlist -> Băm (hash) các kênh -> Lưu vào DB -> Áp dụng Filter (lọc kênh người lớn, lọc theo quốc gia).
2.  **So khớp EPG (Matching):** Sau khi có kênh, hệ thống sử dụng Sentence-Transformers để gợi ý hoặc tự động gán dữ liệu lịch phát sóng (EPG) từ các nguồn XMLTV vào kênh tương ứng.
3.  **Yêu cầu Stream (Streaming):** Client gửi yêu cầu tới `/watch/channel_id` -> Hệ thống kiểm tra `ChannelProfile` (quyền truy cập) -> Kiểm tra số lượng kết nối còn trống trong Redis -> Khởi tạo backend (FFmpeg/Streamlink) -> Proxy luồng dữ liệu về client.
4.  **Ghi hình (DVR):** Người dùng chọn chương trình trên Guide -> Tạo bản ghi `Recording` -> Signal tạo Celery Task với `ETA` (thời gian bắt đầu) -> Đúng giờ, Worker tự động khởi chạy một luồng proxy và lưu dữ liệu vào file `.ts` trên ổ đĩa.
5.  **Giám sát (Monitoring):** Mỗi kết nối stream đều được đăng ký vào Redis. Dashboard frontend sẽ subcribe vào WebSocket để hiển thị biểu đồ băng thông và danh sách người dùng đang xem thời gian thực.

### Tổng kết
Dispatcharr không chỉ là một trình phát IPTV đơn thuần mà là một **Stream Gateway** phức tạp. Nó giải quyết bài toán quản lý hàng nghìn kênh từ nhiều nguồn khác nhau, chuẩn hóa chúng và cung cấp một giao diện quản lý tập trung với khả năng tự động hóa cực cao nhờ sự kết hợp giữa Django và Celery.