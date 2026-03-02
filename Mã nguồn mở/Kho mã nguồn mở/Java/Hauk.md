Dưới đây là bản phân tích chi tiết về dự án **Hauk** - một dịch vụ chia sẻ vị trí thời gian thực tự vận hành (self-hosted).

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology Stack)

Hauk được thiết kế với tiêu chí nhẹ, bảo mật và quyền riêng tư tuyệt đối, thể hiện qua lựa chọn công nghệ:

*   **Backend (PHP):** Sử dụng PHP thuần cho các API xử lý. Điểm đặc biệt là **không sử dụng cơ sở dữ liệu quan hệ (SQL)**. Thay vào đó, nó sử dụng **Memcached** hoặc **Redis** (In-memory storage) để lưu trữ vị trí. Điều này đảm bảo dữ liệu chỉ tồn tại trong RAM và tự động biến mất khi hết hạn session, tối ưu cho quyền riêng tư.
*   **Android App (Java):** Được viết bằng Java truyền thống, sử dụng các thư viện hệ thống của Android để tối ưu hóa việc chạy ngầm (Foreground Service).
*   **Frontend (Web Viewer):** Sử dụng HTML/JS và thư viện bản đồ **Leaflet.js** để hiển thị vị trí của người chia sẻ.
*   **Giao thức truyền tải:** Sử dụng HTTP POST để gửi các gói tin (Packets). Dữ liệu được đóng gói dưới dạng tham số URL-encoded hoặc mã hóa E2E.
*   **Bảo mật:** 
    *   **E2E Encryption:** Sử dụng thuật toán AES-256 (CBC mode) và PBKDF2 để tạo khóa từ mật khẩu.
    *   **Proxy/Tor:** Hỗ trợ kết nối qua proxy SOCKS/HTTP và các tên miền `.onion`.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hauk áp dụng kiến trúc **Ephemeral Relay (Chuyển tiếp tạm thời)**:

1.  **Dữ liệu ngắn hạn:** Hauk coi vị trí là dữ liệu "tạm thời". Kiến trúc này loại bỏ hoàn toàn việc ghi log vị trí vào ổ đĩa. Khi session kết thúc, mọi dấu vết vị trí trong RAM bị xóa bỏ.
2.  **Mô hình Provider-Relay-Viewer:**
    *   **Provider (App):** Thu thập tọa độ từ GPS và đẩy lên Server.
    *   **Relay (Backend):** Nhận dữ liệu, xác thực password/session, và lưu vào bộ nhớ đệm (Cache).
    *   **Viewer (Web):** Truy vấn Server để lấy tọa độ mới nhất và vẽ lên bản đồ.
3.  **Tách biệt Session và Share:** Một **Session** (phiên kết nối) có thể chứa nhiều **Share** (liên kết chia sẻ). Điều này cho phép "Group Share" - nơi nhiều người cùng tham gia vào một mã PIN để thấy nhau.

---

### 3. Các Kỹ thuật Chính (Key Techniques)

*   **Background Location Persistence (Chạy ngầm bền bỉ):** Trong `LocationPushService.java`, Hauk sử dụng `ForegroundService` kết hợp với một `Notification` cố định. Đây là kỹ thuật chuẩn để ngăn Android giết tiến trình khi ứng dụng bị thu nhỏ.
*   **Workarounds cho OEM Power Saving:** Hauk có một module `DeviceChecker.java` cực kỳ chi tiết dành riêng cho các dòng máy như Huawei, Xiaomi, OnePlus. Nó nhận diện thiết bị và hướng dẫn người dùng tắt các trình tối ưu hóa pin của hãng vốn cực kỳ "hung hãn" đối với các app chạy ngầm.
*   **E2E Encryption (Mã hóa đầu cuối):** Trong `LocationUpdatePacket.java`, nếu chế độ E2E được bật, toàn bộ tọa độ (Lat, Lon) và độ chính xác sẽ được mã hóa bằng AES trước khi gửi. Backend chỉ đóng vai trò lưu trữ các chuỗi base64 vô nghĩa; chỉ người có link chứa mật khẩu (frontend) mới giải mã được.
*   **Resumable Sessions (Khôi phục phiên):** Sử dụng `ResumableSessions.java` để lưu trữ thông tin phiên vào `SharedPreferences`. Nếu điện thoại bị khởi động lại hoặc app bị crash, Hauk sẽ hỏi người dùng có muốn tiếp tục phiên chia sẻ cũ hay không.
*   **Broadcast Integration:** Cho phép các ứng dụng bên thứ ba điều khiển Hauk thông qua Android Intent Broadcast (trong `Receiver.java`).

---

### 4. Luồng Hoạt động của Hệ thống (System Workflow)

#### Luồng 1: Khởi tạo Chia sẻ (Session Initiation)
1.  App gửi gói tin đến `create.php` kèm theo mật khẩu server và thời gian chia sẻ (duration).
2.  Server xác thực, tạo một `sessionID` ngẫu nhiên và lưu cấu hình vào Redis/Memcached.
3.  Server trả về URL chia sẻ công khai.

#### Luồng 2: Cập nhật Vị trí (Location Update)
1.  `LocationPushService` lắng nghe sự kiện từ `LocationManager` của Android (GPS/Network).
2.  Khi có tọa độ mới (và thỏa mãn điều kiện khoảng cách tối thiểu/thời gian), App gọi API `post.php`.
3.  API này cập nhật tọa độ vào Redis/Memcached với key là `sessionID`.

#### Luồng 3: Xem Vị trí (Viewing)
1.  Người nhận mở link chia sẻ trên trình duyệt.
2.  Frontend gọi API `fetch.php` định kỳ (polling).
3.  Server truy xuất Redis, nếu session còn hạn, trả về tọa độ mới nhất.
4.  Leaflet.js cập nhật vị trí Marker trên bản đồ.

#### Luồng 4: Kết thúc (Termination)
1.  Khi hết thời gian chia sẻ (TTL trong Redis hết hạn) hoặc người dùng nhấn "Stop" (gọi `stop.php`).
2.  Server xóa dữ liệu session khỏi Cache.
3.  Mọi truy cập vào link chia sẻ sau đó sẽ nhận được thông báo phiên đã kết thúc.

### Tổng kết
Hauk là một ví dụ điển hình về việc xây dựng một hệ thống **Privacy-by-Design**. Nó giải quyết bài toán khó nhất trên Android (duy trì GPS chạy ngầm) và bài toán khó nhất về bảo mật (không lưu trữ dữ liệu nhạy cảm lâu dài) một cách rất thông minh và tinh gọn.