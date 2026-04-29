Dựa trên mã nguồn và cấu trúc thư mục của **WorkAdventure**, đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng metaverse mã nguồn mở này:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

WorkAdventure được xây dựng dưới dạng một **Monorepo** với sự kết hợp của nhiều công nghệ hiện đại:

*   **Ngôn ngữ:** Chủ yếu là **TypeScript** (78%) cho cả Frontend và Backend, đảm bảo tính nhất quán của dữ liệu qua các giao thức.
*   **Game Engine:** **Phaser 3** được dùng để xử lý đồ họa 2D phong cách 16-bit, quản lý va chạm và render nhân vật.
*   **Frontend UI:** **Svelte** được sử dụng rộng rãi cho các lớp giao diện đè lên game (Overlay UI) như menu, chat, và bảng điều khiển nhờ tính nhẹ và hiệu suất cao.
*   **Giao thức giao tiếp:**
    *   **gRPC & Protobuf:** Dùng để định nghĩa và truyền tải tin nhắn giữa các dịch vụ Backend (`back`, `pusher`, `map-storage`) với hiệu năng cực cao.
    *   **WebSockets:** Duy trì kết nối thời gian thực giữa trình duyệt và máy chủ.
*   **Video/Audio Chat:** 
    *   **WebRTC (Peer-to-Peer):** Dùng cho các nhóm nhỏ (mặc định dưới 4 người).
    *   **LiveKit (SFU):** Sử dụng cho quy mô lớn hơn, hỗ trợ ghi hình (recording) và luồng ổn định.
    *   **Jitsi / BigBlueButton:** Tích hợp sẵn cho các phòng họp ảo.
*   **Hạ tầng:** **Docker & Docker Compose** cho phát triển, **Kubernetes (Helm Charts)** cho triển khai quy mô lớn. **Redis** dùng để quản lý state và lưu trữ tạm thời.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của WorkAdventure tập trung vào việc giải quyết bài toán **"Phòng game multiplayer quy mô lớn"**:

*   **Spatial Partitioning (Phân vùng không gian):** Đây là tư duy quan trọng nhất. Bản đồ được chia thành các **Zone** (lưới ô vuông). Hệ thống chỉ gửi dữ liệu cập nhật (vị trí, hành động) của người chơi cho những người khác đang đứng ở các Zone lân cận. Điều này giúp giảm tải băng thông và CPU theo cấp số nhân.
*   **Kiến trúc Microservices phân rã:**
    *   `play`: Frontend xử lý logic game tại client.
    *   `back`: Xử lý logic nghiệp vụ về phòng (Room), nhóm (Group) và các biến số (Variables).
    *   `map-storage`: Dịch vụ riêng biệt để quản lý và phục vụ các tệp tin bản đồ (.tmj, .wam).
    *   `uploader`: Xử lý việc tải lên các tài nguyên (file chat, ảnh).
*   **Strategy Pattern (Chiến lược giao tiếp):** Hệ thống tự động chuyển đổi giữa các chiến lược giao tiếp (WebRTC hoặc LiveKit) dựa trên cấu hình hoặc số lượng người dùng trong một khu vực.
*   **Decoupled Scripting:** Logic của bản đồ không nằm trong mã nguồn chính mà được thực thi thông qua **Scripting API**. Người dùng có thể viết script riêng (nhúng qua Iframe) để tương tác với thế giới game mà không cần can thiệp vào core engine.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Position Notifier & Interest Management:** Kỹ thuật lọc người nghe. Khi người chơi A di chuyển, máy chủ tính toán xem ai đang "quan tâm" đến Zone đó để phát tín hiệu, thay vì broadcast toàn bộ server.
*   **Protobuf Transformers:** Kỹ thuật chuyển đổi tự động giữa các đối tượng JSON và định dạng nhị phân Protobuf để tối ưu hóa việc lưu trữ và truyền tải.
*   **Dead Reckoning / Interpolation:** Phía client sử dụng các thuật toán nội suy để dự đoán chuyển động của nhân vật khác, giúp di chuyển trông mượt mà ngay cả khi mạng có độ trễ (latency).
*   **Dynamic Area Property Management:** Quản lý các thuộc tính vùng (như vùng hội thoại, vùng cấm vào, vùng mở web) một cách linh hoạt thông qua các Command pattern (Create/Update/Delete Area Commands).
*   **OIDC Federation:** Kỹ thuật xác thực liên hợp, cho phép đăng nhập qua nhiều nhà cung cấp định danh (OpenID Connect) và đồng bộ tag người dùng để phân quyền trong game.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Giai đoạn kết nối:** 
    *   Người dùng truy cập URL phòng -> Frontend tải asset từ `maps` và cấu hình từ `map-storage`.
    *   Xác thực qua OIDC -> Lấy JWT token.
    *   Kết nối WebSocket đến dịch vụ `back` qua gRPC thông qua một gateway.
2.  **Giai đoạn di chuyển và tương tác:**
    *   Người chơi di chuyển trên Phaser -> Client gửi tọa độ mới lên `back`.
    *   `back` cập nhật `Zone` của người chơi đó. Nếu người chơi bước vào một **Area** đặc biệt (ví dụ: mở website), một event sẽ được bắn về client để mở Iframe.
3.  **Giai đoạn hội thoại (Proximity Chat):**
    *   Khi hai người chơi bước lại gần nhau (`distance < MINIMUM_DISTANCE`), `back` tạo một **Group** (bong bóng hội thoại).
    *   Client nhận tín hiệu có nhóm mới -> Kích hoạt luồng WebRTC để truyền video/audio trực tiếp giữa các trình duyệt.
4.  **Giai đoạn chỉnh sửa thế giới (Map Editor):**
    *   Người có quyền Admin mở trình chỉnh sửa -> Thực hiện thay đổi tài nguyên bản đồ.
    *   Các thay đổi được đóng gói thành gRPC request gửi đến `map-storage` để lưu vĩnh viễn và đồng bộ ngay lập tức cho các người chơi khác đang trong phòng đó thông qua dịch vụ `back`.

---
**Kết luận:** WorkAdventure là một hệ thống phân tán phức tạp nhưng được tối ưu rất tốt về mặt **hiệu suất không gian**. Việc sử dụng gRPC và chia lưới (Grid-based) khiến nó có khả năng chịu tải hàng nghìn người chơi đồng thời trên một bản đồ lớn.