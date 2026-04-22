Dựa trên cấu trúc thư mục và mã nguồn chi tiết của dự án **GoDoxy (godoxy)**, dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Ngôn ngữ chính:** **Go (Golang)** chiếm hơn 95%. Lựa chọn này tận dụng khả năng xử lý song song (concurrency) cực tốt thông qua Goroutines và tính hiệu suất cao của các thư viện mạng.
*   **Web Framework (API):** **Gin Gonic**. Được dùng để xây dựng REST API cho WebUI quản lý.
*   **JSON Engine:** **Sonic (by ByteDance)**. Sử dụng để phân tích cú pháp JSON với tốc độ cực nhanh, tối ưu cho việc xử lý metrics và log.
*   **Xử lý chứng chỉ:** **Lego (Go ACME client)**. Tự động cấp phát và gia hạn chứng chỉ SSL từ Let's Encrypt qua DNS-01 challenge.
*   **Logging:** **ZeroLog**. Thư viện logging hướng cấu trúc (structured logging) với hiệu suất rất cao.
*   **Mạng & Proxy:**
    *   Sử dụng `net/http` tiêu chuẩn kết hợp với các tùy chỉnh sâu về `ReverseProxy`.
    *   Hỗ trợ **HTTP/3 (QUIC)** qua thư viện `quic-go`.
    *   Hỗ trợ **Proxy Protocol** để bảo toàn IP gốc của client.
*   **Cơ sở dữ liệu GeoIP:** **MaxMind**. Dùng cho tính năng ACL (Access Control List) dựa trên quốc gia và múi giờ.

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của GoDoxy được thiết kế theo hướng **Modular (Mô-đun hóa)** và **Provider-based (Dựa trên nhà cung cấp)**:

*   **Kiến trúc Internal-heavy:** Hầu hết logic nghiệp vụ nằm trong thư mục `internal/`, giúp che giấu implementation chi tiết và chỉ export những gì cần thiết.
*   **Cơ chế Provider:** GoDoxy không chỉ quét Docker cục bộ. Nó coi Docker, Proxmox, hay các tệp cấu hình YAML đều là các "Provider". Điều này cho phép hệ thống mở rộng linh hoạt:
    *   `docker_provider`: Quét label của container.
    *   `proxmox_provider`: Tích hợp với LXC/VM.
    *   `file_provider`: Đọc cấu hình tĩnh từ tệp.
*   **Tách biệt quyền hạn (Security Bundling):** 
    *   `socket-proxy`: Một thành phần riêng biệt bảo vệ Docker Socket, chỉ cho phép các lệnh đọc (hoặc lệnh cụ thể) để tránh container bị chiếm quyền điều khiển host.
    *   `agent`: Chạy trên các node từ xa để GoDoxy điều khiển từ trung tâm qua mTLS.
*   **Thiết kế hướng trạng thái (State-driven):** GoDoxy duy trì một trạng thái toàn cục (Global State) về các route. Khi có bất kỳ thay đổi nào (container khởi động lại, file cấu hình bị sửa), hệ thống sẽ kích hoạt cơ chế **Hot-reload** mà không làm ngắt quãng các kết nối hiện tại.

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **Lock-free Map:** Sử dụng `xsync.Map` để quản lý các cấu hình route và cache. Điều này giúp tăng hiệu suất đọc ghi đồng thời mà không bị "bottleneck" bởi mutex truyền thống.
*   **Lifetime Management (Task System):** Dự án sử dụng một thư viện tùy chỉnh `goutils/task` để quản lý vòng đời của các tiến trình chạy ngầm. Mỗi tiến trình là một `Subtask` có thể được hủy bỏ (Cancel) hoặc dọn dẹp (Cleanup) sạch sẽ khi hệ thống shutdown.
*   **Custom Middleware Chain:** Hệ thống middleware được xây dựng linh hoạt (thư mục `internal/net/gphttp/middleware`). Mỗi request đi qua một chuỗi các bộ lọc: `CloudflareRealIP` -> `ACL` -> `RateLimit` -> `Auth` -> `ForwardAuth`.
*   **Zero-copy & Buffer Pooling:** Sử dụng `synk` (một wrapper cho `sync.Pool`) để tái sử dụng các byte buffer, giảm thiểu áp lực lên bộ dọn rác (GC) khi truyền tải dữ liệu proxy lớn.

### 4. Luồng hoạt động của hệ thống (System Workflows)

#### A. Luồng khởi tạo và Khám phá (Discovery Flow)
1.  **Start:** Load cấu hình từ `config.yml` và biến môi trường.
2.  **Provider Initialization:** Khởi tạo các watcher cho Docker (qua socket) và File (qua fsnotify).
3.  **Route Mapping:** Quét các label (ví dụ: `proxy.aliases`) từ Docker. Chuyển đổi các thông tin thô này thành cấu hình chuẩn (Scheme, Host, Port).
4.  **Security Setup:** Kiểm tra và tự động cấp chứng chỉ SSL nếu cần qua ACME.

#### B. Luồng xử lý yêu cầu (Request Flow)
1.  **Entrypoint:** Client gửi yêu cầu đến cổng 80/443.
2.  **TCP/IP Filter (ACL):** Kiểm tra IP/CIDR/Country của client. Nếu bị chặn, kết nối bị đóng ngay lập tức ở mức TCP.
3.  **TLS Handshake:** Nếu là HTTPS, thực hiện bắt tay. Sử dụng SNI để chọn đúng chứng chỉ.
4.  **Middleware Pipeline:**
    *   Xác định IP thực (Real IP).
    *   Kiểm tra xác thực (OIDC/User-Pass).
    *   Áp dụng các quy tắc tùy chỉnh (Custom Rules).
5.  **Reverse Proxy:** Chuyển tiếp yêu cầu đến backend (Container IP hoặc URL từ xa).

#### C. Luồng Idle-sleep (Tính năng độc đáo)
1.  **Monitoring:** `idlewatcher` theo dõi lưu lượng traffic của từng route.
2.  **Auto-stop:** Nếu một service không có traffic trong X phút, GoDoxy sẽ ra lệnh cho Docker/Proxmox dừng container đó lại để tiết kiệm tài nguyên.
3.  **Wake-on-demand:** Khi có một request mới đến route đang "ngủ", GoDoxy giữ request đó lại (hold), khởi động container lên, chờ cho đến khi healthcheck pass, rồi mới forward request đi.

### 5. Điểm nổi bật
Dự án này vượt xa các reverse proxy thông thường như Nginx hay Caddy ở chỗ nó tích hợp sâu vào **vòng đời của hạ tầng (Infrastructure Lifecycle)**. Việc hỗ trợ cả Docker và Proxmox cùng với cơ chế tiết kiệm điện (Idle-sleep) làm cho nó trở thành một công cụ lý tưởng cho môi trường Homelab và Microservices hiện đại.