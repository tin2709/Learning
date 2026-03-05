Dựa trên nội dung các tệp nguồn của dự án **Beszel**, dưới đây là phân tích chi tiết về kiến trúc, công nghệ và tư duy lập trình của nền tảng giám sát hệ thống này.

---

### 1. Phân tích Công nghệ cốt lõi (Core Tech Stack)

Beszel được xây dựng bằng ngôn ngữ **Go**, tận dụng tối đa hệ sinh thái thư viện hệ thống mạnh mẽ:

*   **Hệ thống & Metrics:** Sử dụng `shirou/gopsutil` để lấy thông số CPU, RAM, Disk, Network. Đây là thư viện tiêu chuẩn trong Go cho việc giám sát hệ thống.
*   **Giao tiếp (Communication):** 
    *   **WebSocket:** Sử dụng `lxzan/gws` (một thư viện WebSocket hiệu năng cao) làm kênh truyền tin chính giữa Agent và Hub.
    *   **SSH:** Tích hợp `gliderlabs/ssh` để cung cấp kênh kết nối dự phòng hoặc thay thế, cho phép Hub kết nối vào Agent qua giao thức SSH.
*   **Định dạng dữ liệu:** Sử dụng **CBOR** (`fxamacker/cbor/v2`) thay vì JSON để truyền tải dữ liệu. CBOR là định dạng nhị phân, giúp giảm kích thước gói tin và tăng tốc độ xử lý (tương tự Protocol Buffers nhưng không cần schema phức tạp).
*   **Cơ sở dữ liệu (Hub):** Hub được xây dựng trên nền **PocketBase** (SQLite nhúng), giúp triển khai cực kỳ đơn giản (chỉ một file thực thi duy nhất).
*   **Container:** Tương tác trực tiếp với **Docker/Podman API** qua Unix Socket hoặc TCP để lấy số liệu thống kê của từng container.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Beszel đi theo mô hình **Hub-Agent** với tư duy tối giản và hiệu quả:

*   **Mô hình "Pull over Push" linh hoạt:** 
    *   Thông thường, Hub sẽ kéo dữ liệu từ Agent.
    *   Tuy nhiên, Beszel hỗ trợ Agent chủ động kết nối ngược lại Hub qua WebSocket (Reverse Connection). Điều này rất hữu ích khi Agent nằm sau NAT hoặc Firewall mà không cần mở port.
*   **Tách biệt Hệ thống (System) và Container:** Kiến trúc tách biệt rõ ràng việc thu thập thông số máy chủ vật lý và các container đang chạy, cho phép theo dõi chi tiết đến từng service.
*   **Khả năng mở rộng (Scalability):** Mỗi Agent là một thực thể độc lập, cực nhẹ. Hub đóng vai trò là nơi tổng hợp dữ liệu (Aggregator) và cung cấp giao diện hiển thị.
*   **Thiết kế hướng nền tảng (Platform-Agnostic):** Mã nguồn xử lý rất kỹ các điều kiện khác nhau giữa Linux, Windows, Darwin (macOS) và FreeBSD thông qua cơ chế `//go:build`.

---

### 3. Các kỹ thuật lập trình then chốt (Key Techniques)

#### a. Quản lý Delta và Caching (Interval-aware Tracking)
Trong tệp `agent_cache.go` và `docker.go`, Beszel sử dụng kỹ thuật tính toán **Delta**:
*   Thay vì chỉ lấy giá trị tức thời, Agent lưu lại snapshot của lần đo trước đó. 
*   Khi có yêu cầu dữ liệu (ví dụ: mỗi 60 giây), nó tính toán sự chênh lệch (Delta) để đưa ra con số chính xác về tốc độ (Bytes/sec, % CPU).
*   **Caching:** Hệ thống cache theo thời gian (`cacheTimeMs`) để nếu nhiều người dùng cùng xem dashboard, Agent không phải truy vấn hệ điều hành liên tục, giảm tải cho CPU.

#### b. Bảo mật dựa trên chữ ký (Signature-based Auth)
Trong `client.go`, Agent không sử dụng mật khẩu truyền thống để xác thực với Hub. Thay vào đó, nó sử dụng **SSH Public Key Authentication**:
*   Hub gửi một thử thách (challenge).
*   Agent kiểm tra chữ ký của Hub dựa trên danh sách Key được cấu hình sẵn. Điều này cực kỳ an toàn và tránh được các cuộc tấn công Brute-force.

#### c. Quản lý GPU đa nền tảng
Beszel có bộ quản lý GPU rất tinh vi (`gpu.go`):
*   Hỗ trợ Nvidia qua `nvidia-smi` hoặc thư viện liên kết động **NVML** (sử dụng `purego` để gọi hàm C mà không cần CGO).
*   Hỗ trợ AMD qua `rocm-smi` hoặc đọc trực tiếp từ `/sys/class/drm` (Linux sysfs).
*   Hỗ trợ Intel và thậm chí cả Apple Silicon (M1/M2/M3) qua `powermetrics`.

#### d. Tối ưu hóa thu thập dữ liệu Disk (Sleeping Disks)
Một chi tiết rất tinh tế là `DISK_USAGE_CACHE`. Agent cho phép cache thông tin sử dụng ổ đĩa trong thời gian dài (ví dụ: 1 giờ) để tránh việc truy vấn liên tục làm ổ đĩa HDD đang ở chế độ ngủ (Standby/Sleep) phải quay lại (spin up), giúp tiết kiệm điện và tăng tuổi thọ ổ cứng.

---

### 4. Luồng hoạt động (Operational Flow)

1.  **Khởi tạo (Startup):** Agent quét cấu hình, kiểm tra các cảm biến có sẵn (Sensors), GPU, Docker socket và tạo một vân tay hệ thống (Fingerprint).
2.  **Kết nối (Connection):** 
    *   Agent cố gắng kết nối WebSocket tới Hub (dựa trên `HUB_URL` và `TOKEN`).
    *   Nếu thất bại, nó khởi động SSH Server và đợi Hub kết nối tới.
3.  **Thu thập dữ liệu (Sampling Loop):**
    *   Agent chạy các vòng lặp ngầm (goroutines) để thu thập dữ liệu từ CPU, RAM, Disk I/O, Network.
    *   Nếu có GPU hoặc Docker, các bộ thu thập tương ứng sẽ được kích hoạt.
4.  **Phản hồi (Communication):**
    *   Khi Hub yêu cầu, Agent lấy dữ liệu từ Cache hoặc thực hiện một lần quét mới.
    *   Dữ liệu được đóng gói bằng **CBOR** và gửi về Hub.
5.  **Cảnh báo (Alerting - tại Hub):** Hub nhận dữ liệu, lưu vào SQLite và so sánh với các ngưỡng (threshold) để gửi thông báo (Telegram, Discord, Email...) thông qua thư viện `shoutrrr`.

### Tổng kết
Beszel là một ví dụ điển hình về việc viết công cụ hệ thống bằng Go: **Cực nhẹ, không phụ thuộc (zero-dependency), và xử lý tốt các đặc thù của phần cứng.** Việc sử dụng CBOR và SSH-auth cho thấy tác giả ưu tiên hiệu suất và bảo mật thực tế hơn là các giao thức web cồng kềnh.