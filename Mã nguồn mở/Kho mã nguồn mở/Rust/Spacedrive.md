Dựa trên các tệp tin và cấu trúc thư mục bạn cung cấp, dưới đây là phân tích chuyên sâu về hệ thống **Spacedrive** - một trình quản lý tệp tin phân tán thế hệ mới.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

Spacedrive là một dự án "Rust-first" với sự kết hợp của các công nghệ hiện đại nhất trong hệ sinh thái này:

*   **Ngôn ngữ lập trình:** 
    *   **Backend:** Rust (chiếm ~77%) xử lý logic lõi, VDFS và networking.
    *   **Frontend:** TypeScript/React kết hợp với Tailwind CSS v4.
    *   **Bridges:** Swift (cho native macOS/iOS) và Kotlin (cho Android).
*   **Hệ thống tệp tin phân tán (VDFS):** Không sử dụng cấu trúc tệp tin truyền thống của OS mà xây dựng một lớp ảo hóa bên trên.
*   **Mạng P2P:** Sử dụng **Iroh** thay vì libp2p. Iroh được xây dựng trên giao thức QUIC, tối ưu cho việc hole-punching (xuyên NAT) và truyền tải dữ liệu trực tiếp giữa các thiết bị mà không cần máy chủ trung tâm.
*   **Định danh nội dung (Content Identity):** Sử dụng thuật toán **BLAKE3** để băm (hashing) nội dung tệp. Điều này cho phép hệ thống nhận diện các tệp tin trùng lặp trên các thiết bị khác nhau dựa vào nội dung thay vì tên hoặc đường dẫn.
*   **Cơ sở dữ liệu:** **SeaORM + sqlx** (với SQLite) cho metadata cục bộ. Để phục vụ tìm kiếm AI/Semantic, hệ thống sử dụng **LanceDB** (Vector Database) và **FastEmbed**.
*   **Xử lý đa phương tiện:** Tích hợp sâu **FFmpeg** cho video, **libheif** cho ảnh iPhone, **Pdfium** cho tài liệu và **Whisper** cho việc chuyển đổi giọng nói thành văn bản cục bộ.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Spacedrive V2 chuyển dịch sang kiến trúc **CQRS (Command Query Responsibility Segregation)** và **DDD (Domain Driven Design)**:

*   **Mô hình Client-Daemon:** Core chạy như một dịch vụ nền (Daemon). Các ứng dụng (Tauri, CLI, Mobile) là các Client kết nối qua Unix Domain Sockets hoặc WebSockets sử dụng JSON-RPC.
*   **SdPath (Universal Addressing):** Đây là tư duy cốt lõi. Một tệp tin được định danh bằng một URI duy nhất (`SdPath`), bất kể nó nằm trên ổ cứng cục bộ, S3, hay trên một máy tính của bạn ở một thành phố khác.
*   **Leaderless Sync:** Không có node "chủ". Hệ thống sử dụng **HLC (Hybrid Logical Clocks)** để đánh dấu thời gian các thay đổi, giúp giải quyết xung đột dữ liệu một cách nhất quán trên tất cả các thiết bị trong mạng P2P.
*   **Kiến trúc Plugin/Extension dựa trên WASM:** Sử dụng WebAssembly để chạy các tiện ích mở rộng (như trình trích xuất metadata cho ảnh) trong môi trường sandbox an toàn, không phụ thuộc vào hệ điều hành.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Type Mirroring (Specta):** Hệ thống sử dụng crate `specta` để tự động tạo ra các định nghĩa Type cho TypeScript và Swift từ các struct Rust. Điều này đảm bảo tính nhất quán tuyệt đối về kiểu dữ liệu giữa Backend và Frontend mà không cần viết thủ công.
*   **Durable Jobs (MessagePack):** Các tác vụ nặng (như quét 10TB dữ liệu) được thiết kế để có thể tạm dừng và phục hồi (Resumable). Trạng thái công việc được tuần tự hóa bằng MessagePack vào database, giúp hệ thống tiếp tục chạy ngay cả khi ứng dụng bị tắt đột ngột.
*   **Transactional Actions:** Mọi thao tác tệp (copy, move) đều trải qua quy trình: **Preview -> Confirm -> Execute**. Người dùng có thể xem trước dung lượng tiết kiệm được hoặc các xung đột tiềm tàng trước khi thực hiện thực tế.
*   **Safety Screening (Prompt Guard 2):** Một kỹ thuật bảo mật AI đặc sắc. Hệ thống quét nội dung tệp cục bộ để phát hiện các cuộc tấn công "Prompt Injection" trước khi dữ liệu đó được đưa vào chỉ mục tìm kiếm cho các AI Agent (Spacebot).

### 4. Luồng Hoạt động Hệ thống (System Operational Flow)

#### A. Luồng Ingest (Nạp dữ liệu)
1.  **Discovery:** Trình theo dõi (Watcher) phát hiện tệp mới.
2.  **Indexing:** Hệ thống tạo một bản ghi "Entry" trong database.
3.  **Content ID:** Tính toán mã băm BLAKE3 (có kỹ thuật lấy mẫu - sampling - cho tệp lớn để tăng tốc).
4.  **Metadata Extraction:** Kích hoạt các Job chạy nền để lấy EXIF, OCR văn bản hoặc tạo Thumbnail.

#### B. Luồng P2P Sync
1.  **Pairing:** Hai thiết bị trao đổi khóa công khai Ed25519 qua QR Code.
2.  **Connection:** Thiết lập kênh QUIC mã hóa qua Iroh.
3.  **Log Replaying:** Thiết bị A gửi các bản ghi thay đổi (HLC-ordered logs) cho thiết bị B.
4.  **Metadata Merge:** Thiết bị B áp dụng các thay đổi vào database cục bộ. Lưu ý: Chỉ metadata được đồng bộ tự động, tệp tin gốc chỉ được truyền tải khi có yêu cầu (On-demand).

#### C. Luồng AI Agent (Spacebot)
1.  **Request:** Người dùng ra lệnh bằng ngôn ngữ tự nhiên.
2.  **Semantic Search:** Hệ thống thực hiện tìm kiếm vector trên LanceDB để tìm ngữ cảnh liên quan.
3.  **Permission Check:** Kiểm tra quyền truy cập của Agent trên thiết bị mục tiêu.
4.  **Proxy Execution:** Lệnh có thể được gửi từ điện thoại nhưng thực thi thực tế trên một server thông qua mạng P2P.

### Tổng kết
**Spacedrive** không chỉ là một trình duyệt tệp; nó là một **hệ điều hành dữ liệu phân tán**. Kỹ thuật lập trình của họ tập trung vào việc xóa nhòa ranh giới giữa các thiết bị vật lý, trong khi vẫn duy trì tính bảo mật tuyệt đối bằng cách xử lý mọi thứ cục bộ (Local-first) và mã hóa đầu cuối (E2EE). Đây là một trong những dự án Rust có kiến trúc phức tạp và hoàn thiện nhất hiện nay.