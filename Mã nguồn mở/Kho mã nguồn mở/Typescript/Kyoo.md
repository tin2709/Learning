Dựa trên nội dung mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Kyoo** - một hệ thống máy chủ truyền thông tự lưu trữ (self-hosted media server).

---

### 1. Công nghệ Cốt lõi (Core Tech Stack)
Kyoo được xây dựng theo kiến trúc **Monorepo** với sự kết hợp của nhiều ngôn ngữ lập trình hiệu năng cao:

*   **API (Backend chính):** TypeScript chạy trên **Bun** (thay vì Node.js) để tối ưu tốc độ. Sử dụng **ElysiaJS** làm web framework và **Drizzle ORM** để tương tác với cơ sở dữ liệu **PostgreSQL**.
*   **Auth (Xác thực):** Viết bằng **Go**, xử lý JWT, OIDC (Google, Discord, Authelia) và quản lý người dùng.
*   **Scanner (Quét dữ liệu):** Viết bằng **Python (FastAPI)**, chịu trách nhiệm theo dõi thư mục phim, phân tích tên file (sử dụng các quy tắc regex phức tạp cho Anime) và lấy metadata từ TMDB/TVDB.
*   **Transcoder (Chuyển mã):** Viết bằng **Go**, tối ưu hóa việc xử lý video bằng FFmpeg, hỗ trợ tăng tốc phần cứng (Nvidia, Intel VA-API, QSV).
*   **Front (Giao diện):** Sử dụng **React Native & Expo**, cho phép chạy trên cả trình duyệt Web và ứng dụng Android với cùng một cơ sở mã.
*   **Infrastructure:** Sử dụng **Docker & Docker Compose** làm nền tảng triển khai chính, **Traefik** làm API Gateway và **Meilisearch** cho tính năng tìm kiếm.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Mindset)
Kyoo đi theo một lộ trình khác biệt so với Plex hay Jellyfin:

*   **Microservices Approach:** Chia nhỏ hệ thống thành các dịch vụ độc lập (Containerized). Điều này giúp dễ dàng mở rộng, ví dụ: bạn có thể chạy nhiều container Transcoder trên các máy khác nhau để chia sẻ tải.
*   **Stateless & Scalable:** Không dựa vào SQLite (như Plex) mà dùng PostgreSQL, giúp xử lý đồng thời tốt hơn và tránh hỏng dữ liệu khi tải cao.
*   **API Gateway Offloading:** Sử dụng Traefik để điều phối traffic và xử lý xác thực (Authentication) ngay tại Gateway thông qua dịch vụ Auth trước khi gửi request đến API hoặc Transcoder.
*   **Philosophy "Setup once, forget about it":** Tư duy thiết kế không yêu cầu người dùng phải đặt tên file theo chuẩn khắt khe hay cấu trúc thư mục cố định. Hệ thống tự động nhận diện phim ngay cả với tên file "lạ" của các nhóm sub Anime.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Features)

*   **Dynamic Transcoding:** Chuyển mã video theo thời gian thực (HLS). Điểm đặc biệt là khả năng "seek" (tua) ngay lập tức mà không cần chờ transcoder xử lý xong toàn bộ đoạn video.
*   **Anime Name Parsing:** Bộ lọc mạnh mẽ có khả năng bóc tách thông số như tập phim, độ phân giải, nhóm sub, định dạng âm thanh (Opus, FLAC...) từ những tên file dài và phức tạp.
*   **Watch List Scrubbing:** Tự động đồng bộ trạng thái xem với các dịch vụ bên thứ ba như SIMKL, giúp người dùng không phải đánh dấu thủ công từng tập đã xem.
*   **OIDC Integration:** Hỗ trợ đăng nhập một lần thông qua các nhà cung cấp định danh phổ biến, phù hợp cho việc quản lý người dùng trong gia đình hoặc cộng đồng nhỏ.
*   **Enhanced Subtitle Support:** Hỗ trợ đầy đủ định dạng sub phức tạp (SSA/ASS) và tận dụng font chữ nhúng sẵn trong file video để hiển thị đúng ý đồ của người làm sub.

---

### 4. Luồng hoạt động của hệ thống (Workflow)

1.  **Giai đoạn Quét (Scanning):**
    *   Dịch vụ `Scanner` theo dõi thư mục video thông qua hệ thống file của OS.
    *   Khi có file mới, nó phân tích tên file và gửi yêu cầu đến TMDB/TVDB để lấy thông tin phim/series.
    *   Dữ liệu sau đó được đẩy về `API` để lưu vào PostgreSQL.
2.  **Giai đoạn Truy cập (Accessing):**
    *   Người dùng mở `Front` (Web/App), gửi request qua `Traefik`.
    *   `Traefik` hỏi `Auth` xem user này có quyền không.
    *   Nếu hợp lệ, `API` trả về danh sách phim kèm hình ảnh (đã được xử lý Blurhash để hiển thị mượt mà).
3.  **Giai đoạn Phát (Streaming):**
    *   Khi nhấn Play, trình phát yêu cầu luồng từ `Transcoder`.
    *   `Transcoder` đọc file gốc từ ổ đĩa, kiểm tra cấu hình phần cứng (Nvidia/Intel) và bắt đầu băm nhỏ video thành các đoạn HLS.
    *   Tiến độ xem được gửi liên tục về `API` để cập nhật `History` và `Watchlist`.

### Tóm tắt
Kyoo là một dự án **hiện đại, chú trọng vào hiệu suất và sự tự động hóa**. Việc chọn Bun và Go cho thấy tác giả ưu tiên tốc độ xử lý và khả năng xử lý song song. Đây là lựa chọn tuyệt vời cho những người dùng muốn có trải nghiệm như Netflix nhưng tự quản lý dữ liệu trên máy chủ cá nhân mà không muốn tốn thời gian dọn dẹp thư viện thủ công.