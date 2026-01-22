Dựa trên mã nguồn và tài liệu của dự án **Databag**, dưới đây là phân tích chi tiết về mặt kỹ thuật, kiến trúc và luồng hoạt động của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology)

Databag là một ứng dụng Full-stack hiện đại, được tối ưu hóa để chạy trên các thiết bị phần cứng yếu (như Raspberry Pi Zero).

*   **Backend (Server):** Sử dụng ngôn ngữ **Golang**. Lựa chọn này giúp hệ thống đạt hiệu suất cao, tiêu tốn rất ít RAM và CPU.
*   **Database:** **SQLite**. Đây là lựa chọn thông minh cho việc tự host (self-hosting) vì không cần cài đặt máy chủ database phức tạp (như MySQL/PostgreSQL), dữ liệu gói gọn trong một file.
*   **Frontend (Web & Mobile):**
    *   **Web:** ReactJS với bộ build Tool là Vite/Yarn.
    *   **Mobile:** React Native (hỗ trợ cả iOS và Android) dùng chung logic với bản Web thông qua một bộ **SDK** riêng.
*   **Truyền tải dữ liệu & Real-time:**
    *   **REST API:** Cho các tác vụ thông thường.
    *   **WebSockets:** Để đẩy thông báo (push events) ngay lập tức mà không cần polling, giúp giảm độ trễ.
    *   **WebRTC:** Sử dụng giao thức STUN/TURN để thực hiện cuộc gọi âm thanh và hình ảnh trực tiếp giữa các thiết bị.
*   **Xử lý Media:** Sử dụng **FFmpeg** (cho video/audio) và **ImageMagick** (cho hình ảnh) để tạo thumbnail và nén dữ liệu trước khi lưu trữ.

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Databag xoay quanh ba trụ cột: **Phi tập trung (Decentralized)**, **Liên bang (Federated)** và **Bảo mật tối đa (Privacy-First)**.

*   **Kiến trúc Liên bang (Federation):** Tương tự như Email hoặc Mastodon. Người dùng ở máy chủ (node) A có thể tìm kiếm và nhắn tin cho người dùng ở máy chủ B. Mỗi "node" là độc lập nhưng có khả năng giao tiếp với nhau.
*   **Định danh dựa trên mật mã (Crypto-based Identity):** Danh tính người dùng không gắn liền với số điện thoại hay email, mà dựa trên cặp **Khóa Công khai - Khóa Bí mật (Public-Private Key)**. Điều này giúp người dùng có toàn quyền kiểm soát danh tính mà không phụ thuộc vào một nhà cung cấp trung tâm.
*   **Cơ chế "Sealed Topics" (Chủ đề được niêm phong):** Đây là tư duy về mã hóa đầu cuối (E2EE). Khi một chủ đề được "Sealed", máy chủ chỉ đóng vai trò lưu trữ các khối dữ liệu đã mã hóa. Chỉ những người tham gia có khóa bí mật mới có thể giải mã nội dung.
*   **Tư duy tối giản (Lightweight):** Mọi thành phần từ Docker Image (dùng Alpine Linux) đến cách viết code Go đều hướng tới việc giảm thiểu dấu chân (footprint) hệ thống.

### 3. Các kỹ thuật then chốt (Key Techniques)

*   **E2EE & Key Exchange:** Sử dụng thuật toán **RSA** để trao đổi khóa và **AES** để mã hóa nội dung tin nhắn. Quá trình mã hóa/giải mã diễn ra hoàn toàn ở phía Client (trình duyệt hoặc App di động).
*   **Native Crypto trên Mobile:** Trên ứng dụng di động, Databag sử dụng các thư viện Native (như `react-native-rsa-native`) để thực hiện các thao tác xử lý khóa với tốc độ nhanh hơn so với JavaScript thuần.
*   **UnifiedPush:** Kỹ thuật hỗ trợ thông báo đẩy mà không phụ thuộc hoàn toàn vào Google (FCM) hay Apple (APN), cho phép người dùng sử dụng các dịch vụ push tự host hoặc mã nguồn mở.
*   **Transcoding Scripts:** Hệ thống có một bộ các script shell (`transform_vhd.sh`, `transform_ithumb.sh`...) để tự động xử lý các tệp tin media được tải lên, đảm bảo chúng có thể hiển thị tốt trên mọi thiết bị.
*   **Relay Server (TURN):** Để vượt qua các lớp NAT (tường lửa mạng), Databag tích hợp khả năng cấu hình máy chủ Relay, cho phép gọi điện video ngay cả khi hai thiết bị nằm trong các mạng riêng biệt.

### 4. Tóm tắt luồng hoạt động (Project Workflow)

Dưới đây là luồng hoạt động chính khi người dùng tương tác với dự án:

#### A. Luồng Đăng ký & Định danh:
1.  **Admin** tạo một "Account Token" từ giao diện quản trị.
2.  **Người dùng** sử dụng Token này để đăng ký tài khoản trên Node.
3.  **Client** tự động tạo cặp khóa RSA. Khóa công khai được gửi lên server để định danh, khóa bí mật được lưu an toàn trên thiết bị người dùng (hoặc bảo vệ bằng mật khẩu).

#### B. Luồng Kết nối Liên bang (Federation):
1.  Người dùng A tìm kiếm B qua địa chỉ `username@domain.com`.
2.  Node A liên hệ với Node B để lấy thông tin Profile và Khóa công khai của B.
3.  Hai bên thực hiện quy trình "Handshake" để xác nhận quan hệ bạn bè.

#### C. Luồng Nhắn tin (Messaging):
1.  **Gửi:** Client soạn tin nhắn -> Nếu là Sealed Topic, Client mã hóa bằng AES -> Gửi lên Server qua API Go.
2.  **Lưu trữ:** Server Go nhận tin nhắn, lưu vào SQLite và gọi script xử lý nếu có tệp đính kèm.
3.  **Nhận:** Server thông báo cho Client nhận qua WebSocket hoặc Push Notification.
4.  **Giải mã:** Client nhận nhận dữ liệu mã hóa -> Dùng khóa bí mật để giải mã và hiển thị lên UI.

#### D. Luồng Triển khai (Deployment):
1.  **Docker:** `Dockerfile` thực hiện build đa giai đoạn (Multi-stage build). Giai đoạn 1 build React Web, giai đoạn 2 build Go binary, cuối cùng gộp lại thành một Image siêu nhỏ gọn.
2.  **Entrypoint:** File `entrypoint.sh` khởi tạo các thư mục cần thiết, cấu hình SQLite và chạy server Go trên port 7000.

### Kết luận
Databag là một ví dụ điển hình của việc kết hợp giữa sức mạnh hệ thống của **Go** và sự linh hoạt của **React/React Native**. Dự án này không chỉ là một công cụ nhắn tin, mà là một giải pháp kiến trúc hoàn chỉnh cho việc truyền thông bảo mật, đề cao tính tự chủ của người dùng trong kỷ nguyên số.