Dựa trên nội dung mã nguồn của dự án **MiroTalk P2P**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Backend:** Node.js với Framework **Express**.
*   **Signaling (Báo hiệu):** Sử dụng **Socket.io** để điều phối việc thiết lập kết nối giữa các người dùng (Peer).
*   **Real-time Communication:** **WebRTC** (Peer-to-Peer) cho phép truyền tải video, âm thanh và dữ liệu trực tiếp giữa các trình duyệt mà không thông qua server trung gian (giảm độ trễ và chi phí server).
*   **Security:**
    *   **JWT (JSON Web Token):** Xác thực người dùng và bảo vệ phòng họp.
    *   **Crypto-js (AES):** Mã hóa thêm một lớp cho Payload của token.
    *   **DOMPurify & JSDOM:** Chống tấn công XSS (Cross-Site Scripting) bằng cách làm sạch dữ liệu đầu vào.
*   **Frontend:** Vanilla JavaScript (JS thuần), CSS3, HTML5 (Không sử dụng framework nặng như React/Angular để tối ưu tốc độ).
*   **DevOps:** Docker, Docker Compose, Kubernetes (K8s), Shell scripts cho việc tự động hóa cài đặt.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Techniques)

#### a. Kiến trúc Signaling (Báo hiệu)
Mặc dù video/audio đi trực tiếp giữa các máy khách (P2P), nhưng MiroTalk cần một server trung gian để các máy khách "tìm thấy nhau". Tư duy ở đây là **Thin Server**: Server chỉ làm nhiệm vụ kết nối ban đầu, sau đó "đứng sang một bên" để các Peer tự nói chuyện.

#### b. Xử lý âm thanh nâng cao (Advanced Audio Processing)
Dự án sử dụng **AudioWorklet** (một kỹ thuật xử lý âm thanh luồng thấp của Web Audio API):
*   `noiseSuppressionProcessor.js`: Tích hợp **RNNoise** (WASM) để khử nhiễu môi trường ngay trên trình duyệt.
*   `volumeProcessor.js`: Tính toán mức âm lượng (VAD - Voice Activity Detection) để hiển thị thanh âm lượng.

#### c. HTML Injection & SEO/Metadata Management
Dự án có file `htmlInjector.js`. Kỹ thuật này cho phép server thay đổi các thẻ Meta (OG tags) động trước khi gửi file HTML tĩnh về trình duyệt. Điều này giúp việc chia sẻ link lên mạng xã hội hiển thị đúng tiêu đề/hình ảnh của từng phòng họp.

#### d. Quản lý cấu hình linh hoạt
Toàn bộ tính năng (Bật/tắt Chat, Whiteboard, Ghi âm...) được điều khiển qua file `.env` và `config.js`. Tư duy kiến trúc này giúp người dùng dễ dàng **White-label** (đổi thương hiệu) dự án mà không cần can thiệp sâu vào code lõi.

---

### 3. Các kỹ thuật chính nổi bật

1.  **Mã hóa 2 lớp:** Token không chỉ được ký bằng JWT mà còn được mã hóa AES (trong `tokenManager.js`), đảm bảo ngay cả khi lộ JWT secret, thông tin bên trong vẫn an toàn.
2.  **Xử lý File WebM:** Sử dụng `fixWebmDuration.js` để sửa lỗi mất thời lượng (duration) khi ghi video trên trình duyệt (một lỗi phổ biến của MediaRecorder API).
3.  **Tích hợp đa nền tảng:** Có sẵn webhook và API hỗ trợ tích hợp vào **Slack**, **Mattermost**, và các ứng dụng bên thứ ba thông qua Iframe API.
4.  **Chống Path Traversal:** Kiểm tra nghiêm ngặt tên phòng họp để tránh người dùng truy cập trái phép vào các đường dẫn hệ thống (`validate.js`).

---

### 4. Tóm tắt luồng hoạt động (Project Workflow)

1.  **Khởi tạo (Initialization):**
    *   Người dùng truy cập trang chủ, hệ thống tải cấu hình thương hiệu (`brand.js`).
    *   Người dùng nhập tên phòng. Hệ thống kiểm tra tính hợp lệ (XSS/Path Traversal).

2.  **Thiết lập kết nối (Signaling):**
    *   Trình duyệt kết nối tới Server qua Socket.io.
    *   Nếu phòng yêu cầu mật khẩu hoặc bảo vệ chủ phòng (Host Protection), server sẽ kiểm tra JWT/OIDC.
    *   Server gửi thông tin máy chủ **STUN/TURN** để giúp các trình duyệt xuyên thủng NAT/Firewall.

3.  **Trao đổi Media (P2P Handshake):**
    *   Peer A gửi "Offer" (SDP) qua server báo hiệu tới Peer B.
    *   Peer B gửi lại "Answer".
    *   Hai bên trao đổi ứng viên ICE (ICE Candidates) để tìm đường đi ngắn nhất đến nhau.

4.  **Truyền tải và Tính năng trong cuộc gọi:**
    *   Khi kết nối P2P được thiết lập, Video/Audio/DataChannel bắt đầu truyền trực tiếp.
    *   **Whiteboard:** Dữ liệu vẽ được gửi qua DataChannel tới tất cả mọi người.
    *   **Chat/File Sharing:** Cũng được truyền qua DataChannel để đảm bảo tính riêng tư (không qua server).
    *   **Khử nhiễu:** Luồng âm thanh được chạy qua `nodeProcessor.js` (RNNoise) trước khi gửi đi.

5.  **Kết thúc:**
    *   Khi người dùng rời phòng, socket ngắt kết nối, server thông báo cho các Peer còn lại để xóa bỏ element video tương ứng.
    *   Người dùng được điều hướng đến trang khảo sát (Survey) nếu được cấu hình.

---
**Kết luận:** Đây là một dự án có kiến trúc rất bài bản cho ứng dụng WebRTC hiện đại, chú trọng vào việc xử lý phía Client (Frontend-heavy) để giảm tải cho Server và tối ưu hóa tính riêng tư cho người dùng.