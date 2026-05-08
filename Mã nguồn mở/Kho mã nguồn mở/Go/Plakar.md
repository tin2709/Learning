Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Plakar**, đây là phân tích chuyên sâu về hệ thống quản trị dữ liệu và sao lưu mã nguồn mở này:

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

*   **Ngôn ngữ lập trình:** Sử dụng **Go 1.25**, tận dụng các tính năng mới nhất của ngôn ngữ để tối ưu hiệu suất và quản lý bộ nhớ.
*   **Storage Engine (Kloset):** Plakar được xây dựng bên trên **Kloset**, một công cụ lưu trữ dữ liệu bất biến (immutable data store). Điểm mạnh là khả năng **Deduplication (Khử trùng lặp)** mạnh mẽ dựa trên CDC (Content Defined Chunking), giúp tiết kiệm dung lượng khi lưu trữ nhiều phiên bản sao lưu.
*   **Mật mã học (Cryptography):** Hệ thống đã trải qua kiểm định (audit) và sử dụng các thuật toán hiện đại nhất:
    *   **Argon2id:** Để băm mật khẩu (KDF).
    *   **AES-GCM-SIV:** Mã hóa các khối dữ liệu (chunks), chống lại việc lạm dụng nonce.
    *   **AES-KW (Key Wrap):** Mã hóa các khóa phụ (subkeys).
    *   **BLAKE3:** Dùng cho hashing và mã xác thực thông điệp (MAC) nhờ tốc độ cực nhanh.
*   **Giao thức & Truyền tải:** Sử dụng **Msgpack** để tuần tự hóa dữ liệu nội bộ và **gRPC** (thông qua `integration-grpc`) cho giao tiếp giữa các thành phần hoặc plugin.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Plakar đã tiến hóa từ mô hình Agent sang mô hình **CLI-centric với tiến trình hỗ trợ (cached)**:

*   **Tiến trình `cached`:** Đây là một kiến trúc đặc sắc. `cached` là một tiến trình chạy nền nhẹ nhàng, chuyên biệt cho việc duy trì bộ nhớ đệm (cache) và quản lý khóa (locking). Điều này giúp CLI thực thi nhanh hơn mà không cần khởi động lại toàn bộ trạng thái cache mỗi lần chạy lệnh.
*   **Abstraction Layer (Connectors):** Hệ thống tách biệt hoàn toàn giữa logic sao lưu và hạ tầng lưu trữ thông qua các interface: `importer` (nguồn dữ liệu), `exporter` (đích khôi phục), và `storage` (nơi lưu trữ vật lý như S3, SFTP, Local FS).
*   **VFS (Virtual File System):** Plakar coi mỗi bản sao lưu (snapshot) là một hệ thống tệp ảo. Người dùng có thể duyệt, tìm kiếm hoặc gắn (mount) các snapshot này vào hệ điều hành thông qua **FUSE**.
*   **Snapshot-based:** Mọi thứ trong Plakar đều xoay quanh "Snapshot". Mỗi snapshot chứa toàn bộ ngữ cảnh (metadata, cấu trúc cây tệp) tại một thời điểm nhất định.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Nhúng giao diện Web (Embedded UI):** Plakar nhúng toàn bộ frontend (Vue/TypeScript) vào trong tệp thực thi duy nhất bằng `go:embed`. Lệnh `plakar ui` khởi chạy một máy chủ HTTP nội bộ để cung cấp giao diện quản trị trực quan.
*   **Hệ thống Plugin động:** Sử dụng `pkg.go` để quản lý các "Integrations". Plakar có thể tải và thực thi các connector bên thứ ba mà không cần biên dịch lại mã nguồn chính, thông qua cơ chế quản lý gói riêng.
*   **Xử lý đa nền tảng:** Tách biệt logic xử lý file và lock giữa Unix (`flock`) và Windows (`LockFileEx`), đảm bảo tính nhất quán của dữ liệu trên mọi hệ điều hành.
*   **URL Signing & JWT:** Trong `api/api_snapshot.go`, Plakar sử dụng JWT để ký (sign) các URL tải dữ liệu. Kỹ thuật này cho phép tạo ra các liên kết tải xuống tạm thời, an toàn mà không cần duy trì session phức tạp.
*   **TUI (Terminal UI):** Ngoài giao diện dòng lệnh truyền thống, Plakar tích hợp TUI dựa trên `bubbletea`, cung cấp trải nghiệm đồ họa ngay trong terminal cho các tác vụ dài hơi.

### 4. Luồng Hoạt động Hệ thống (System Operational Flow)

#### A. Luồng Sao lưu (Backup Flow)
1.  **Duyệt (Scan):** CLI quét các thư mục nguồn thông qua `importer`.
2.  **Chia nhỏ & Khử trùng (Chunking):** Dữ liệu được chia thành các khối bằng CDC. Khối nào đã tồn tại trong repository (dựa trên mã băm) sẽ được bỏ qua.
3.  **Mã hóa:** Các khối dữ liệu mới được mã hóa bằng AES-GCM-SIV tại máy khách (Client-side encryption).
4.  **Tải lên:** Chuyển dữ liệu đã mã hóa lên `storage`.
5.  **Commit:** Tạo bản ghi Snapshot chứa cấu trúc cây tệp và lưu vào Repository.

#### B. Luồng Khôi phục (Restore Flow)
1.  **Truy vấn Metadata:** CLI lấy thông tin cấu trúc tệp từ snapshot.
2.  **Lấy khối (Fetch Chunks):** Xác định các khối dữ liệu cần thiết.
3.  **Giải mã:** Khối dữ liệu được giải mã bằng khóa bí mật từ phía người dùng.
4.  **Tái tạo:** Ghi dữ liệu ra đích thông qua `exporter`.

#### C. Luồng Đồng bộ (Sync Flow)
*   Sử dụng lệnh `sync` để sao chép dữ liệu giữa các Repository (ví dụ từ Local sang S3). Nhờ deduplication, chỉ những khối dữ liệu bị thiếu ở đích mới được truyền đi.

### Tổng kết
**Plakar** là một giải pháp sao lưu hiện đại, tập trung vào **hiệu suất cực cao** (nhờ BLAKE3 và CDC) và **bảo mật tuyệt đối** (Zero-knowledge encryption). Kiến trúc tách biệt giữa CLI và tiến trình duy trì cache (`cached`) là một điểm sáng giúp hệ thống này vượt trội hơn các công cụ sao lưu truyền thống về mặt trải nghiệm người dùng trên các tập dữ liệu lớn.