Dựa trên cấu trúc thư mục, tệp tin và mã nguồn bạn cung cấp từ kho lưu trữ **Jitsi Meet**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của dự án này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **WebRTC (Web Real-Time Communication):** Đây là "linh hồn" của Jitsi. Jitsi sử dụng thư viện `lib-jitsi-meet` (được nhắc đến trong `package.json` và `Makefile`) để xử lý việc truyền tải video/audio, mã hóa và kết nối P2P hoặc qua Server (SFU).
*   **React & React Native:** Dự án sử dụng một codebase duy nhất nhưng đa nền tảng.
    *   **Web:** React với TypeScript.
    *   **Mobile:** React Native (Android/iOS).
    *   **Cơ chế:** Sử dụng các hậu tố tệp như `.web.ts`, `.native.ts`, `.any.ts` để phân tách logic giữa các nền tảng (xem trong `tsconfig.json`).
*   **XMPP (Extensible Messaging and Presence Protocol):** Jitsi sử dụng giao thức XMPP (thông qua server Prosody) để làm lớp tín hiệu (Signaling). Các plugin trong thư mục `resources/prosody-plugins` cho thấy việc quản lý phòng họp, xác thực, và tin nhắn đều qua XMPP.
*   **WebAssembly (Wasm):** Sử dụng để xử lý các tác vụ hiệu năng cao ngay trên trình duyệt:
    *   `rnnoise.wasm`: Khử tiếng ồn.
    *   `tflite.wasm` & TensorFlow: Làm mờ/thay đổi nền ảo (Virtual Background).
*   **Redux:** Quản lý trạng thái toàn cục cực kỳ phức tạp của một cuộc gọi (danh sách người tham gia, trạng thái mic/cam, quyền hạn...).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc hướng tính năng (Feature-Driven Architecture):**
    *   Mọi thứ nằm trong `react/features/`. Mỗi tính năng (ví dụ: `chat`, `recording`, `authentication`) là một module độc lập chứa: `actions`, `reducer`, `middleware`, `components`.
    *   Cách tiếp cận này giúp dự án có hơn 80 tính năng mà vẫn có thể bảo trì và mở rộng dễ dàng.
*   **Cơ chế Registry (Registry Pattern):**
    *   Thay vì một file Redux khổng lồ, Jitsi sử dụng `ReducerRegistry` và `MiddlewareRegistry`. Mỗi tính năng tự đăng ký logic của mình với hệ thống trung tâm. Điều này giảm thiểu xung đột mã nguồn khi nhiều người cùng phát triển.
*   **Tính trừu tượng hóa nền tảng (Platform Abstraction):**
    *   Jitsi xây dựng các "Abstract Components". Logic nghiệp vụ (Business Logic) được viết một lần, trong khi giao diện hiển thị (UI) được tùy biến riêng cho Web (HTML/CSS) và Mobile (Native Components).
*   **Kiến trúc Middleware-Heavy:**
    *   Jitsi đẩy hầu hết logic xử lý sự kiện RTC (ví dụ: khi một người khác tắt mic) vào Redux Middleware. Component chỉ việc lắng nghe state và hiển thị, giúp UI luôn mượt mà.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Selective Forwarding Unit (SFU):** Dù đây là mã nguồn Client, nhưng kiến trúc được thiết kế để làm việc với Jitsi Videobridge. Thay vì gửi 1 luồng đến từng người (Mesh), client chỉ gửi 1 luồng duy nhất lên server và nhận về các luồng từ người khác.
*   **Xử lý luồng âm thanh/hình ảnh bằng AI:**
    *   Tích hợp **Face Landmarks** (nhận diện khuôn mặt) và **Emotion Detection** ngay trên luồng video của client.
*   **Hệ thống Bundle tinh gọn:**
    *   `webpack.config.js` chia nhỏ ứng dụng thành nhiều bundle: `app.bundle` (chính), `external_api` (cho phép nhúng Jitsi vào web khác), và các `workers` xử lý dưới nền.
*   **Tối ưu hóa Native (Android/iOS):**
    *   Sử dụng `ConnectionService` (Android) và `CallKit` (iOS) để tích hợp cuộc gọi vào hệ thống của điện thoại (hiển thị như một cuộc gọi viễn thông thông thường).
    *   Kỹ thuật `Picture-in-Picture` (PiP) được xử lý sâu ở mức native (`PictureInPictureModule.java`).

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi tạo (Initialization):**
    *   Người dùng truy cập URL -> `app.js` khởi chạy -> React mount vào DOM.
    *   Hệ thống kiểm tra cấu hình (`config.js`, `interface_config.js`) và ngôn ngữ.
2.  **Kết nối (Connection):**
    *   `lib-jitsi-meet` thiết lập kết nối XMPP đến Server.
    *   Người dùng gia nhập một MUC (Multi-User Chat) - đây chính là "phòng họp" ảo.
3.  **Thiết lập Media (Media Setup):**
    *   Trình duyệt yêu cầu quyền truy cập Camera/Mic.
    *   Tạo các "Local Tracks" (luồng dữ liệu cục bộ). Áp dụng các hiệu ứng (khử nhiễu, nền ảo) qua Wasm.
4.  **Trao đổi tín hiệu & Kết nối RTC:**
    *   Thông qua XMPP, các client trao đổi file SDP (Session Description) để hiểu cách kết nối với nhau hoặc với Bridge.
    *   Thiết lập PeerConnection để truyền tải media thực tế.
5.  **Duy trì trạng thái (State Sync):**
    *   Bất kỳ thay đổi nào (ai đó vào phòng, ai đó giơ tay) đều gửi qua kênh tín hiệu (DataChannel hoặc XMPP).
    *   Sự kiện này kích hoạt Redux Action -> Cập nhật Store -> React UI thay đổi tương ứng.

### Kết luận
Dự án **Jitsi Meet** là một hình mẫu về việc sử dụng **React/Redux** để quản lý một hệ thống thời gian thực cực kỳ phức tạp. Sự kết hợp giữa **WebRTC** cho truyền tải, **XMPP** cho điều khiển và **Wasm** cho xử lý tín hiệu giúp Jitsi vừa mạnh mẽ như phần mềm native, vừa linh hoạt như một trang web.