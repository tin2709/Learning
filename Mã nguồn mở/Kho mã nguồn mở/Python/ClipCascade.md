Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp cho dự án **ClipCascade**, dưới đây là phân tích chuyên sâu về hệ thống đồng bộ hóa clipboard đa nền tảng này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

ClipCascade là một hệ thống phân tán (Distributed System) phức tạp, sử dụng các công nghệ hiện đại nhất cho từng nền tảng:

*   **Server (Backend):**
    *   **Java 21 & Spring Boot 3:** Tận dụng Virtual Threads và hiệu năng của JRE mới nhất.
    *   **ActiveMQ:** Làm Message Broker trung gian cho giao thức STOMP (Peer-to-Server).
    *   **Spring Security & JPA:** Xử lý xác thực, phân quyền và quản lý dữ liệu (H2 cho bản nhẹ, PostgreSQL cho đa người dùng).
    *   **Ehcache:** Sử dụng để lưu trữ và quản lý dữ liệu bảo vệ chống Brute-Force (BFA).
*   **Desktop Client (Python):**
    *   **Python 3:** Ngôn ngữ chính xử lý logic nghiệp vụ.
    *   **Tkinter/Custom CLI:** Đa dạng giao diện từ GUI đến terminal (phù hợp cho cả Server Linux).
    *   **PyWin32, Pasteboard, Gtk/xclip:** Các thư viện cấp thấp để can thiệp trực tiếp vào hệ thống Clipboard của Windows, macOS và Linux.
    *   **Aiortc:** Thư viện xử lý WebRTC cho chế độ truyền dữ liệu trực tiếp Peer-to-Peer (P2P).
*   **Mobile Client (Android):**
    *   **React Native:** Framework chính cho UI.
    *   **Kotlin (Native Modules):** Dùng để xử lý các tác vụ nền (Foreground Service), theo dõi Logcat và can thiệp sâu vào hệ thống Clipboard của Android.
*   **Bảo mật & Mã hóa:**
    *   **AES-GCM (256-bit):** Mã hóa dữ liệu Clipboard đầu cuối (End-to-End).
    *   **PBKDF2:** Hàm dẫn xuất khóa (Key Derivation) từ mật khẩu người dùng.
    *   **SHA3-512:** Băm mật khẩu phía Client trước khi gửi lên Server để tăng tính bảo mật tuyệt đối.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế với tư duy **Hybrid Connectivity (Kết nối hỗn hợp)**:

*   **Mô hình P2S (Peer-to-Server):** Sử dụng STOMP qua WebSocket. Server đóng vai trò trung tâm nhận và phân phối dữ liệu. Ưu điểm: Tin cậy, dễ dàng xuyên qua NAT/Firewall.
*   **Mô hình P2P (Peer-to-Peer):** Sử dụng WebRTC Data Channels. Server chỉ đóng vai trò Signaling (bắt tay). Dữ liệu clipboard truyền trực tiếp giữa các thiết bị. Ưu điểm: Băng thông không giới hạn, độ trễ cực thấp, giảm tải cho Server.
*   **Kiến trúc Đa tầng (Layered Architecture):**
    *   **Core Layer:** Chứa logic cấu hình, hằng số và quản lý Cipher.
    *   **Interface Layer:** Trừu tượng hóa kết nối WebSocket (`WSInterface`) để dễ dàng chuyển đổi giữa P2S và P2P.
    *   **Platform-specific Layer:** Tách biệt logic xử lý clipboard cho từng hệ điều hành khác nhau.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Xử lý Clipboard Event-Driven & Polling:**
    *   Trên Windows: Sử dụng `AddClipboardFormatListener` (Win32 API) để nhận thông báo thay đổi thay vì kiểm tra liên tục.
    *   Trên macOS/Wayland: Sử dụng kỹ thuật Polling (0.3s - 1s) kết hợp kiểm tra `changeCount` để tối ưu hóa tài nguyên.
*   **Kỹ thuật "Logcat Monitoring" (Android 10+ Bypass):** Đây là điểm sáng kỹ thuật. Do Android hạn chế quyền đọc clipboard khi ứng dụng chạy ngầm, ClipCascade theo dõi log hệ thống (`READ_LOGS`). Khi phát hiện lỗi sao chép, nó kích hoạt một "Floating Activity" trong chớp mắt để lấy tiêu điểm (Focus) và đọc clipboard một cách hợp lệ.
*   **Payload Fragmentation:** Dữ liệu lớn (ảnh, file) được chia nhỏ thành các mảnh 15KiB để truyền qua WebSocket mà không làm treo buffer hoặc bị ngắt kết nối đột ngột.
*   **Mã hóa End-to-End (E2EE):** Server không giữ khóa giải mã. Khóa được tạo ra từ mật khẩu người dùng + Salt thông qua PBKDF2 ngay tại thiết bị. Dữ liệu trên đường truyền chỉ là byte code vô nghĩa với bên thứ ba.

### 4. Luồng hoạt động hệ thống (System Workflows)

#### Luồng gửi dữ liệu (Outbound):
1.  **Monitor:** Clipboard Manager phát hiện thay đổi (qua event hoặc polling).
2.  **Filter:** Kiểm tra xem nội dung mới có khác nội dung cũ (hash check) và có nằm trong giới hạn kích thước không.
3.  **Process:** Nếu là ảnh/file, chuyển đổi sang Base64.
4.  **Encrypt:** Mã hóa bằng AES-GCM với khóa từ mật khẩu người dùng.
5.  **Transmit:** 
    *   *P2S:* Gửi qua STOMP tới Server -> Server broadcast tới các thiết bị khác của cùng user.
    *   *P2P:* Gửi trực tiếp qua WebRTC Data Channel tới các thiết bị đang kết nối trong "phòng".

#### Luồng nhận dữ liệu (Inbound):
1.  **Receive & Reassemble:** Nhận các mảnh dữ liệu (fragments) và ghép lại thành payload hoàn chỉnh.
2.  **Decrypt:** Sử dụng khóa nội bộ để giải mã AES-GCM.
3.  **Validation:** Kiểm tra hash để tránh vòng lặp đồng bộ (Sync Loop).
4.  **Action:** 
    *   Văn bản/Ảnh: Ghi trực tiếp vào clipboard hệ thống.
    *   File: Lưu vào thư mục tạm và hiển thị thông báo/nút tải xuống cho người dùng.

#### Luồng bảo vệ Brute-Force:
1.  Người dùng đăng nhập thất bại -> Server ghi nhận IP và Username vào `UserAccessTracker`.
2.  Nếu vượt ngưỡng (`CC_MAX_ATTEMPTS_PER_IP`), IP bị khóa tạm thời.
3.  Thời gian khóa tăng dần theo hệ số (`Scaling Factor`) cho mỗi lần vi phạm tiếp theo.

### Tổng kết
**ClipCascade** không chỉ là một ứng dụng copy-paste. Nó là một giải pháp hạ tầng truyền tải dữ liệu thời gian thực được tối ưu hóa cho quyền riêng tư và hiệu năng, giải quyết triệt để các rào cản kỹ thuật của các hệ điều hành hiện đại (đặc biệt là Android).