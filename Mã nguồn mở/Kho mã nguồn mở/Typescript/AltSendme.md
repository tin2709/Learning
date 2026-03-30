Dựa trên mã nguồn của dự án **AltSendme**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Dự án là một ứng dụng Desktop/Mobile đa nền tảng kết hợp giữa hiệu năng của Rust và sự linh hoạt của Web:

*   **Tauri v2 (Framework chính):** Sử dụng để xây dựng giao diện bằng web (React/TS) và logic hệ thống bằng Rust. Tauri giúp giảm dung lượng ứng dụng so với Electron và cung cấp khả năng bảo mật cao hơn.
*   **Iroh (`iroh` & `iroh-blobs`):** Đây là "linh hồn" của ứng dụng. Iroh là một stack networking P2P hiện đại thay thế cho WebRTC/libp2p:
    *   **QUIC & TLS 1.3:** Đảm bảo toàn bộ dữ liệu được mã hóa đầu cuối (E2E) và vận chuyển qua UDP để tối ưu tốc độ.
    *   **NAT Traversal:** Sử dụng kỹ thuật hole-punching để xuyên qua tường lửa/NAT và fallback về Relay server (n0) khi cần.
    *   **BLAKE3 Hashing:** Dùng để định danh dữ liệu (content-addressing) và kiểm tra tính toàn vẹn cực nhanh.
*   **Frontend Stack:** React 18, TypeScript, Tailwind CSS v4 (sử dụng engine mới nhất), Framer Motion (hiệu ứng), và Zustand để quản lý trạng thái.
*   **Thumbnail Engine:** Sử dụng `image` (Rust) cho ảnh và các API native (AVFoundation trên macOS, GStreamer/FFMPEG trên Linux/Windows) để trích xuất khung hình video làm ảnh preview.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của AltSendme tập trung vào tính **phi tập trung** và **quyền riêng tư**:

*   **Tách biệt logic lõi (Core Separation):** 
    *   Phần `sendme/` là một thư viện Rust thuần túy, không phụ thuộc vào GUI, có thể chạy độc lập. 
    *   Phần `src-tauri/` đóng vai trò là "Orchestrator" (người điều phối), kết nối thư viện lõi với giao diện người dùng.
*   **Cơ chế Metadata riêng biệt:** Thay vì nhét toàn bộ thông tin file (tên, size, ảnh preview) vào "Ticket" (khiến ticket rất dài), kiến trúc sư đã thiết kế một giao thức Metadata riêng (`METADATA_ALPN`). Người nhận sẽ dùng Ticket để kết nối và yêu cầu Metadata trước khi quyết định tải file về.
*   **Kiến trúc Local-first:** Dữ liệu không bao giờ nằm trên máy chủ trung gian. Ứng dụng biến máy tính của người gửi thành một "Provider node" tạm thời.
*   **Quản lý tài nguyên bằng RAII:** Trong Rust, việc duy trì kết nối P2P phụ thuộc vào việc giữ cho `Router` và `TempTag` (trong `ShareHandle`) còn sống. Nếu các đối tượng này bị Drop, server sẽ tự đóng, đảm bảo không rò rỉ tài nguyên.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Custom Protocol Handler:** Triển khai trait `ProtocolHandler` của Iroh để định nghĩa cách xử lý luồng dữ liệu riêng cho Metadata. Kỹ thuật này cho phép ứng dụng mở rộng tính năng networking P2P rất linh hoạt.
*   **Async/Await & Concurrency:** Sử dụng `tokio` runtime và `FuturesUnordered` để xử lý đồng thời nhiều file/luồng trong một thư mục mà không làm treo ứng dụng.
*   **Bridge Pattern (Tauri Commands):** Sử dụng các Command để giao tiếp giữa JavaScript và Rust. Đặc biệt là việc bọc `AppHandle` của Tauri vào một trait `EventEmitter` để Core logic có thể bắn sự kiện (như tiến trình tải) về UI mà không bị phụ thuộc ngược vào framework Tauri.
*   **Platform-specific abstraction:** Sử dụng `#[cfg(target_os = "...")]` dày đặc để tối ưu hóa cho từng hệ điều hành (ví dụ: tạo Context Menu trên Windows Registry, hoặc dùng Cocoa API trên macOS).
*   **Thumbnail Generation:** Sử dụng `spawn_blocking` để đẩy các tác vụ xử lý ảnh nặng nề ra khỏi luồng async chính, tránh gây lag cho việc truyền tải file.

### 4. Luồng hoạt động hệ thống (System Operational Flow)

#### A. Luồng Gửi (Sending Flow):
1.  **Chuẩn bị:** Người dùng thả file -> Rust tính toán kích thước, MIME type và tạo thumbnail -> Lưu tạm vào Iroh Store local.
2.  **Khởi tạo Node:** Một node Iroh được dựng lên với các ALPN (Blobs và Metadata).
3.  **Ticket:** Hệ thống tạo một chuỗi "Ticket" chứa: Node ID (định danh thiết bị), Relay URL và Hash của dữ liệu.
4.  **Lắng nghe:** Ứng dụng duy trì trạng thái chờ (Wait) cho đến khi có máy khách kết nối.

#### B. Luồng Nhận (Receiving Flow):
1.  **Handshake Metadata:** Người nhận dán Ticket -> Ứng dụng kết nối tới máy gửi qua `METADATA_ALPN` -> Tải thông tin file và thumbnail về hiển thị preview.
2.  **Tải file (Confirmed):** Nếu người dùng nhấn "Accept" -> Ứng dụng kết nối qua giao thức Blobs của Iroh.
3.  **Truyền tải:** QUIC thực hiện hole-punching để tạo kết nối trực tiếp (Direct). Dữ liệu được tải về theo từng chunk, kiểm tra hash BLAKE3 ngay lập tức.
4.  **Kết thúc:** Sau khi tải xong vào store tạm -> Iroh Export dữ liệu ra thư mục Downloads của người dùng -> Xóa bỏ dữ liệu trong store tạm và đóng node.

### Tổng kết
AltSendme là một ví dụ điển hình về việc ứng dụng **Rust P2P Stack** để giải quyết bài toán chia sẻ file mà không cần server. Hệ thống xử lý rất tốt các vấn đề phức tạp về mạng (NAT, Firewalls) và tối ưu hóa trải nghiệm người dùng thông qua việc preview metadata trước khi tải.