

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Hệ thống được xây dựng theo mô hình **Monolith-ish Architecture** nhưng có khả năng điều khiển các tiến trình hệ thống và container bên ngoài rất mạnh mẽ.

*   **Backend (Golang):**
    *   Sử dụng **Go 1.25** (phiên bản rất mới), tận dụng tính năng concurrency (goroutines) để xử lý proxy và giám sát thời gian thực.
    *   **Thư viện quan trọng:** `go-chi` (Routing HTTP), `docker/docker` (Điều khiển Docker Engine qua Socket), `go-acme/lego` (Quản lý chứng chỉ SSL/TLS), `rclone` (Quản lý lưu trữ đám mây).
*   **Frontend (React + Vite):**
    *   Giao diện hiện đại sử dụng **React 18**, **Material UI (MUI)** và **Ant Design Icons**.
    *   Quản lý trạng thái bằng **Redux Toolkit**.
    *   Biểu đồ giám sát (Monitoring) sử dụng **ApexCharts**.
*   **Database:** **MongoDB** (được cấu hình chạy trong container hoặc chế độ "Puppet Mode" để Cosmos tự quản lý).
*   **Networking & Security:**
    *   **Nebula:** Tạo mạng Mesh VPN (Constellation) để kết nối các thiết bị từ xa.
    *   **SmartShield:** Một bộ lọc logic tự viết bằng Go để chống DDoS, Brute-force và Bot.
*   **Storage & Backup:**
    *   **Restic:** Thực hiện sao lưu mã hóa, tăng trưởng (incremental).
    *   **MergerFS & SnapRAID:** Quản lý gộp ổ đĩa và bảo vệ dữ liệu (Parity).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Cosmos không chỉ là một Dashboard mà là một **Secure Gateway**. Tư duy thiết kế xoay quanh 3 trụ cột:

1.  **Zero-Trust Proxy:** Mọi yêu cầu đi vào (Inbound) đều phải qua bộ lọc SmartShield trước khi đến ứng dụng đích. Cosmos không tin tưởng các ứng dụng chạy phía sau nó (giả định ứng dụng có thể có lỗ hổng như Plex leak).
2.  **Container Orchestration (Non-Lock-in):** Khác với Unraid hay CasaOS cố gắng "che giấu" Docker, Cosmos cho phép bạn quản lý container từ UI hoặc trực tiếp qua CLI/Portainer mà không làm hỏng cấu hình hệ thống. Nó sử dụng Docker Label làm "nguồn chân lý" để nhận diện ứng dụng.
3.  **Local-First & Privacy:** Toàn bộ dữ liệu (Logs, Metrics, Auth) nằm trên máy chủ của người dùng. Ngay cả khi Cloudflare (nếu dùng Proxy) bị tấn công, lớp bảo vệ nội bộ của Cosmos vẫn duy trì tính toàn vẹn cho mạng LAN.

---

### 3. Các Kỹ thuật Chính (Key Technical Implementations)

#### a. Kỹ thuật SmartShield (Dynamic Protection)
Thay vì dùng các quy tắc Firewall tĩnh (như iptables), Cosmos thực hiện:
*   **Adaptive Throttling:** Tự động làm chậm (delay) phản hồi nếu một IP gửi quá nhiều request thay vì ngắt kết nối ngay lập tức, giúp phân biệt người dùng thật và bot.
*   **Privileged Groups:** Cho phép các nhóm người dùng cụ thể (như Admin) vượt qua các hạn chế băng thông/request khi hệ thống đang bị tấn công.

#### b. Quản lý Chứng chỉ HTTPS (Lego Integration)
Hệ thống tích hợp thư viện **Lego** mạnh mẽ, hỗ trợ hàng chục nhà cung cấp DNS (Cloudflare, GoDaddy, Google...) để giải quyết **DNS Challenge**. Điều này cho phép tạo Wildcard Certificate (`*.domain.com`) mà không cần mở cổng 80/443.

#### c. Constellation (VPN Mesh)
Sử dụng công nghệ của Slack (Nebula) để tạo mạng LAN ảo. Kỹ thuật này cho phép:
*   Bỏ qua CGNAT (khi nhà mạng không cho mở cổng).
*   Mã hóa đầu cuối (End-to-end encryption) giữa điện thoại và server mà không qua server trung gian của bên thứ ba.

#### d. Kỹ thuật Backup với Restic
Cosmos không chỉ copy file đơn thuần. Nó sử dụng Restic để:
*   **Deduplication:** Nếu bạn có 10 bản backup của cùng một file, nó chỉ lưu 1 lần.
*   **Auto-stop:** Có tùy chọn tự động dừng container để đảm bảo tính toàn vẹn dữ liệu (Snapshot) trước khi backup và khởi động lại ngay sau đó.

---

### 4. Luồng Hoạt động của Hệ thống (System Flow)

#### Luồng Truy cập của Người dùng (User Request Flow):
1.  **Request** đi vào cổng 80/443.
2.  **Cosmos Proxy (Go)** tiếp nhận -> Kiểm tra IP blacklist (Geo-blocking/Bot detection).
3.  **SmartShield** kiểm tra quota (Băng thông, số lượng request/giây).
4.  **Auth Module** kiểm tra Session/Cookie hoặc chuyển hướng đến trang Login (nếu route yêu cầu xác thực).
5.  Nếu OK, request được **Reverse Proxy** chuyển tiếp đến Docker Container nội bộ hoặc một IP khác trong mạng LAN/VPN.

#### Luồng Quản lý Container (Management Flow):
1.  Admin thực hiện thao tác trên **React UI**.
2.  **Vite Proxy** chuyển tiếp API call đến Go Backend.
3.  Go Backend thực hiện kiểm tra quyền (Sudo Mode).
4.  Gửi lệnh qua **Unix Socket** (`/var/run/docker.sock`) để Docker Engine thực thi.
5.  Cosmos lắng nghe **Docker Events** để cập nhật trạng thái UI ngay lập tức mà không cần reload trang.

### 5. Đánh giá Ưu/Nhược điểm từ góc độ Code
*   **Ưu điểm:** Codebase rất hiện đại, sử dụng Go giúp hiệu suất Proxy cực cao. Việc tích hợp sẵn Rclone và Restic biến nó thành một giải pháp "tất cả trong một" thực thụ cho NAS.
*   **Nhược điểm:** Do can thiệp sâu vào hệ thống (Privileged mode, mount host root), nếu Cosmos có lỗ hổng bảo vệ chính mình, kẻ tấn công sẽ có quyền root trên toàn bộ máy chủ. Tuy nhiên, dự án giải quyết điều này bằng cách tách biệt `cosmos-launcher` và tiến trình chính.

**Kết luận:** Cosmos Server là một dự án có tư duy kỹ thuật xuất sắc, kết hợp giữa khả năng quản trị hệ thống Linux truyền thống và sự linh hoạt của Docker hiện đại.