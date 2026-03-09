Dựa trên mã nguồn và cấu trúc thư mục của dự án **n.eko**, dưới đây là phân tích chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Technologies)

N.eko là một hệ thống phức tạp kết hợp giữa ảo hóa cấp độ ứng dụng và truyền tải nội dung thời gian thực.

*   **Backend (Server):** Sử dụng **Go (Golang)**. Lựa chọn này tận dụng khả năng xử lý song song mạnh mẽ và hiệu suất cao của Go để quản lý các luồng dữ liệu (video/audio) và kết nối đồng thời.
*   **Frontend (Client):** Sử dụng **Vue.js** kết hợp với **TypeScript**. Giao diện được thiết kế để xử lý luồng WebRTC và truyền các sự kiện điều khiển (chuột, phím) với độ trễ thấp nhất.
*   **Truyền tải (Streaming):** **WebRTC** là công nghệ chủ đạo. Khác với VNC hay RDP truyền thống dùng ảnh tĩnh, WebRTC cho phép truyền video/audio mượt mà như một cuộc gọi video, tối ưu hóa cho băng thông và độ trễ.
*   **Ảo hóa & Container:** Dựa hoàn toàn vào **Docker**. Mỗi "phòng" là một container Linux chạy server XOrg (X11).
*   **Quản lý hệ thống:** 
    *   **Xvfb/Xorg (Dummy driver):** Tạo ra một màn hình ảo trong bộ nhớ.
    *   **PulseAudio:** Xử lý và bắt luồng âm thanh từ ứng dụng.
    *   **Supervisord:** Quản lý vòng đời của các tiến trình bên trong container (browser, X server, Neko server).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của n.eko được thiết kế theo mô hình **"Browser-as-a-Service"**:

*   **Tách biệt Runtime và Ứng dụng:** Dự án chia làm 2 phần rõ rệt: `runtime/` (môi trường nền tảng gồm drivers, thư viện hệ thống) và `apps/` (các ứng dụng cụ thể như Firefox, Chrome, VLC). Điều này cho phép mở rộng thêm bất kỳ ứng dụng Linux nào một cách dễ dàng.
*   **Điều khiển tập trung (Centralized Control):** Mặc dù nhiều người có thể xem cùng lúc, nhưng hệ thống có cơ chế "Host" (người cầm lái). Kiến trúc hỗ trợ việc chuyển giao quyền điều khiển chuột/phím giữa các thành viên trong phòng.
*   **Hỗ trợ tăng tốc phần cứng (Hardware Acceleration):** Kiến trúc cho phép tùy chọn sử dụng GPU (NVIDIA/Intel) để encode video thông qua các Dockerfile chuyên biệt (`Dockerfile.nvidia`, `Dockerfile.intel`), giúp giảm tải cho CPU và tăng độ mượt.
*   **Plugin System:** Hỗ trợ mở rộng tính năng (như Chat, truyền file) thông qua hệ thống plugin ở cả mức Server và Client.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **X11 Scraper & Injector:** Neko không chỉ chụp màn hình; nó sử dụng một driver tùy chỉnh (`neko_drv.so`) để lấy dữ liệu từ frame buffer của XOrg và đưa ngược các sự kiện chuột/phím từ trình duyệt web vào hệ thống X11 của Linux.
*   **WebRTC Stack (Pion):** Sử dụng thư viện *Pion* (Go) để tự triển khai WebRTC stack. Kỹ thuật này cho phép can thiệp sâu vào việc mã hóa video (VP8/H.264) và âm thanh (Opus) trực tiếp từ nguồn hệ thống.
*   DRM & Widevine:** Sử dụng script `widevinecdm.sh` để nạp module bản quyền giúp xem được các dịch vụ như Netflix, Disney+ ngay trong container Docker. DRMs thường rất khó chạy trong môi trường ảo hóa.
*   **Low Latency Input:** Sử dụng kỹ thuật xử lý sự kiện dựa trên tọa độ tương đối/tuyệt đối từ Client và ánh xạ chúng chính xác vào độ phân giải của màn hình ảo trong container.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi tạo:** Khi container Docker chạy, `supervisord` khởi động X Server ảo, PulseAudio và ứng dụng đích (ví dụ: Firefox).
2.  **Kết nối:** Người dùng truy cập qua trình duyệt. Một kết nối **WebSocket** được thiết lập để trao đổi tín hiệu (Signaling).
3.  **Bắt tay (Handshake):** Client và Server thực hiện bắt tay WebRTC (SDP/ICE) để thiết lập luồng truyền tải trực tiếp.
4.  **Truyền tải (Downstream):**
    *   **Video:** Server "quét" màn hình X11, mã hóa thành luồng video và đẩy qua WebRTC đến Client.
    *   **Audio:** PulseAudio bắt âm thanh từ ứng dụng, Server mã hóa thành luồng Opus và gửi đi.
5.  **Điều khiển (Upstream):** Khi người dùng di chuyển chuột hoặc gõ phím trên giao diện web, Client gửi các tọa độ/mã phím qua DataChannel (WebRTC) hoặc WebSocket. Server nhận được và mô phỏng (inject) các sự kiện này vào X Server ảo.
6.  **Tương tác nhóm:** Server đồng bộ trạng thái giữa tất cả người dùng trong phòng (ai đang điều khiển, danh sách thành viên, tin nhắn chat).

**Kết luận:** n.eko là một giải pháp đột phá cho việc cộng tác từ xa và giải trí nhóm. Nó biến một ứng dụng desktop Linux thành một ứng dụng web đa người dùng với hiệu suất gần như tương đương máy cục bộ nhờ tối ưu hóa WebRTC và Docker.