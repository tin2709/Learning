Dựa trên các tệp tin mã nguồn bạn cung cấp, dưới đây là phân tích chi tiết về dự án **PairDrop** (một bản fork cải tiến từ Snapdrop) theo các khía cạnh công nghệ, kiến trúc và quy trình hoạt động.

---

### 1. Công nghệ Cốt lõi (Core Technology Stack)

PairDrop được xây dựng trên triết lý tối giản (Vanilla) nhưng tận dụng tối đa các API hiện đại của trình duyệt:

*   **Frontend:**
    *   **Vanilla HTML5 / JS (ES6+) / CSS3:** Không sử dụng framework nặng (như React/Angular), giúp tốc độ tải cực nhanh.
    *   **WebRTC (Web Real-Time Communication):** Công nghệ chủ chốt để truyền dữ liệu ngang hàng (P2P) trực tiếp giữa các thiết bị mà không đi qua máy chủ.
    *   **WebSockets:** Dùng cho quá trình "Signaling" (báo hiệu) – giúp các thiết bị tìm thấy nhau và thiết lập kết nối WebRTC ban đầu.
    *   **IndexedDB API:** Sử dụng để lưu trữ các "shared secrets" (bí mật dùng chung) cho tính năng ghép đôi thiết bị vĩnh viễn (Persistent Pairing).
    *   **PWA (Progressive Web App):** Cho phép cài đặt ứng dụng trên điện thoại/máy tính, hỗ trợ Web Share Target API (để xuất hiện trong menu "Share" của hệ điều hành).

*   **Backend:**
    *   **Node.js & Express:** Máy chủ trung gian để phục vụ tệp tĩnh và quản lý các kết nối WebSocket.
    *   **WS Library:** Xử lý các thông điệp báo hiệu giữa các máy khách (peers).

*   **Infrastructure:**
    *   **Docker & Docker Compose:** Đóng gói ứng dụng dễ dàng triển khai.
    *   **Coturn (STUN/TURN Server):** Hỗ trợ truyền dữ liệu khi các thiết bị nằm sau các lớp NAT phức tạp (như mạng công ty, 4G) mà P2P thuần túy không làm được.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Philosophy)

Kiến trúc của PairDrop tập trung vào **"Sự đơn giản cực đoan" (Radical Simplicity)** và **Quyền riêng tư**:

*   **Mô hình Signaling (Báo hiệu):** Máy chủ không bao giờ chạm vào tệp tin của người dùng. Nó chỉ đóng vai trò "người môi giới" để hai thiết bị trao đổi địa chỉ IP và thông tin cấu hình kết nối. Sau khi kết nối P2P được thiết lập, máy chủ hoàn toàn đứng ngoài cuộc.
*   **Hệ thống Phòng (Room System):**
    *   **IP-based Room:** Mặc định các thiết bị có cùng IP công cộng sẽ nhìn thấy nhau (cùng mạng LAN).
    *   **Secret-based Room:** Các thiết bị đã ghép đôi (Pairing) sẽ trao đổi một chuỗi bí mật 256-bit được lưu trong IndexedDB để tìm thấy nhau dù ở khác mạng.
    *   **Public Link Room:** Sử dụng mã 5 chữ số để tạo phòng tạm thời.
*   **Cơ chế Fallback (Dự phòng):** Nếu WebRTC bị chặn (bởi VPN hoặc Firewall), PairDrop có tùy chọn chuyển sang truyền dữ liệu qua WebSocket (dữ liệu đi qua máy chủ - cần được quản trị viên kích hoạt).

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

1.  **Persistent Device Pairing:** Khác với Snapdrop (mất kết nối khi đóng tab), PairDrop dùng IndexedDB để lưu định danh thiết bị. Điều này cho phép "nhận diện người quen" mọi lúc mọi nơi.
2.  **Xử lý tệp tin lớn trên iOS:** Mã nguồn có đoạn xử lý giới hạn bộ nhớ của iOS (chặn gửi > 200MB một lúc hoặc cảnh báo) để tránh trình duyệt Safari bị sập.
3.  **Tối ưu hiệu suất (OffscreenCanvas):** Hiệu ứng vòng tròn chuyển động ở nền được xử lý bằng `OffscreenCanvas` trong `Worker` (nếu trình duyệt hỗ trợ), giúp giải phóng luồng chính (Main Thread) cho việc truyền dữ liệu.
4.  **Hỗ trợ đa nền tảng qua CLI & Scripts:** Dự án cung cấp cả công cụ dòng lệnh (bash script) và tích hợp vào menu chuột phải của Windows (Send to) hoặc Linux (Nautilus scripts).
5.  **Web Share Target API:** Cho phép PairDrop nhận file trực tiếp từ các ứng dụng khác trên Android giống như một ứng dụng bản địa.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow Summary)

Dưới đây là luồng hoạt động khi bạn mở PairDrop để gửi file:

1.  **Khởi tạo (Initialization):**
    *   Trình duyệt tải HTML/CSS/JS. `Service Worker` được đăng ký để hỗ trợ chạy ngoại tuyến và nhận file.
    *   Ứng dụng kiểm tra IndexedDB để lấy các thiết bị đã ghép đôi trước đó.

2.  **Kết nối Signaling (Connection):**
    *   Client thiết lập kết nối WebSocket tới máy chủ Node.js.
    *   Máy chủ xác định IP của Client và đưa vào một "phòng" ảo. Nếu Client gửi kèm "Secret", máy chủ sẽ đưa họ vào phòng riêng với các thiết bị có cùng bí mật đó.

3.  **Khám phá (Discovery):**
    *   Máy chủ gửi danh sách các thiết bị hiện có trong cùng phòng (IP, tên thiết bị, icon động vật).
    *   Giao diện hiển thị các biểu tượng thiết bị tương ứng.

4.  **Thiết lập truyền tải (Handshake):**
    *   Người dùng chọn file và nhấn vào icon người nhận.
    *   Client gửi một yêu cầu (Request) qua WebSocket tới người nhận.
    *   Nếu người nhận nhấn "Accept", hai bên trao đổi thông tin ICE Candidates (địa chỉ mạng) để thiết lập kênh **WebRTC Data Channel**.

5.  **Truyền dữ liệu (Transfer):**
    *   File được chia nhỏ thành các đoạn (chunks) cỡ 64KB.
    *   Dữ liệu được mã hóa và gửi trực tiếp giữa 2 trình duyệt.
    *   Thanh tiến trình (Progress bar) cập nhật trên cả hai màn hình.

6.  **Hoàn tất (Completion):**
    *   Sau khi nhận đủ các đoạn dữ liệu, trình duyệt người nhận sẽ ghép lại thành file hoàn chỉnh và kích hoạt lệnh tải xuống (Download).

---

### Kết luận
PairDrop là một ví dụ xuất sắc về việc sử dụng **WebRTC** để giải quyết vấn đề chia sẻ file mà không cần hạ tầng máy chủ tốn kém. Nó cải thiện đáng kể điểm yếu của các ứng dụng web truyền thống bằng cách sử dụng **PWA** và **IndexedDB** để tạo ra trải nghiệm giống như phần mềm cài đặt sẵn (Native App) trên cả điện thoại và máy tính.