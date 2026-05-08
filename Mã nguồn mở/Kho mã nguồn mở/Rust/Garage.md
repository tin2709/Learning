Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Garage**, đây là một hệ thống lưu trữ đối tượng (Object Storage) phân tán, tương thích với S3, được viết hoàn toàn bằng **Rust**.

Dưới đây là phân tích chuyên sâu về hệ thống này:

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

*   **Ngôn ngữ Rust & Async Runtime:** Dự án sử dụng **Rust** làm ngôn ngữ chính (chiếm 95.1%), tận dụng tối đa tính an toàn bộ nhớ và hiệu suất cao. Luồng xử lý bất đồng bộ dựa trên **Tokio stack** (`hyper` cho HTTP, `tokio` cho IO/Network).
*   **Storage Engine đa dạng:** Garage không cố định một cơ sở dữ liệu. Nó hỗ trợ các adapter cho metadata (`garage_db`):
    *   **LMDB (qua crate `heed`):** Mặc định, cho hiệu suất cực cao.
    *   **SQLite:** Cho tính bền vững và dễ quản lý.
    *   **Fjall:** Một LSM-tree engine hiện đại viết bằng Rust.
*   **Giao thức truyền tải (Network & RPC):** 
    *   Sử dụng hệ thống RPC tùy chỉnh (`garage_rpc`) thay vì các framework nặng nề như gRPC.
    *   **Bảo mật mặc định:** Mọi lưu lượng trong cụm (cluster) đều được mã hóa thông qua `kuska-handshake` (giao thức Secret Handshake của Secure Scuttlebutt), đảm bảo không có dữ liệu truyền đi dưới dạng văn bản thuần túy.
*   **API:** Hỗ trợ đầy đủ **S3 API** và một API thử nghiệm là **K2V** (Key-to-Value) cho phép lưu trữ dữ liệu có cấu trúc phân tán.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Garage chịu ảnh hưởng mạnh mẽ từ bài báo **Amazon Dynamo** và các hệ thống phân tán thế hệ mới:

*   **Kiến trúc Peer-to-Peer (Masterless):** Không có nút điều phối (Master node). Mọi nút trong cụm đều có vai trò như nhau, giúp loại bỏ điểm yếu duy nhất (Single Point of Failure).
*   **Dynamo Ring & Consistent Hashing:** Dữ liệu được phân mảnh và định vị trên một vòng tròn băm (Ring). Việc gán dữ liệu cho các nút dựa trên "vị trí" của chúng trên vòng tròn này, cho phép thêm/bớt nút mà không phải di chuyển toàn bộ dữ liệu.
*   **Cơ chế đồng thuận không dựa trên Raft/Paxos:** Thay vì sử dụng các thuật toán đồng thuận mạnh (thường chậm và khó triển khai trong môi trường geo-distributed), Garage sử dụng:
    *   **CRDTs (Conflict-free Replicated Data Types):** Để đồng bộ Metadata (như tên bucket, key). Điều này cho phép hệ thống hoạt động ngay cả khi bị phân tách mạng (Network Partition).
    *   **Quorum (R+W > N):** Đảm bảo tính nhất quán khi đọc/ghi tệp tin.
*   **Geo-Distribution (Phân tán địa lý):** Garage được thiết kế để chạy trên nhiều trung tâm dữ liệu với độ trễ mạng cao. Nó ưu tiên tính sẵn sàng (Availability) và khả năng chịu lỗi phân đoạn mạng (Partition Tolerance) - theo định lý CAP.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Thiết kế Workspace (Modular Crates):** Mã nguồn được chia thành hơn 10 crates nhỏ (`garage_net`, `garage_block`, `garage_table`,...). Kỹ thuật này giúp quản lý sự phụ thuộc (dependencies) chặt chẽ, giảm thời gian biên dịch và giúp kiểm thử từng phần (unit test) dễ dàng hơn.
*   **Anti-Entropy bằng Merkle Trees:** Trong crate `garage_table`, hệ thống sử dụng cây Merkle để phát hiện sự sai khác dữ liệu giữa các nút một cách nhanh chóng mà không cần truyền toàn bộ dữ liệu qua mạng.
*   **Mã hóa & Nén:** Hỗ trợ nén dữ liệu bằng **Zstd** ở mức block, giúp tiết kiệm dung lượng lưu trữ đáng kể trong khi vẫn giữ tốc độ xử lý nhanh.
*   **Fuzz Testing:** Dự án có một thư mục `fuzz/` riêng biệt, sử dụng `cargo-fuzz` để kiểm thử tự động các cấu trúc dữ liệu CRDT phức tạp, đảm bảo các thuật toán hợp nhất dữ liệu (merge) luôn chính xác.
*   **Abstruction Layer cho Database:** Cách thiết kế `src/db/` với các trait giúp hệ thống có thể chuyển đổi engine lưu trữ chỉ bằng cách thay đổi cấu hình mà không phải sửa logic nghiệp vụ.

### 4. Luồng Hoạt động Hệ thống (System Operational Flow)

Một yêu cầu ghi dữ liệu (Put Object) diễn ra như sau:

1.  **Tiếp nhận API:** `garage_api_s3` nhận request qua HTTP Hyper.
2.  **Định tuyến Metadata:** Hệ thống tra cứu `garage_table` để xác định bucket và quyền truy cập thông qua các bản ghi CRDT.
3.  **Phân mảnh Block:** Tệp tin lớn được chia thành các block nhỏ (`garage_block`). Mỗi block được băm (hash) để kiểm tra tính toàn vẹn.
4.  **Xác định Nút lưu trữ:** Dựa trên Ring, hệ thống xác định 3 nút (theo replication factor mặc định) sẽ lưu giữ các block này.
5.  **Ghi Quorum:** Dữ liệu được gửi đến cả 3 nút qua RPC mã hóa. Chỉ cần 2 trong 3 nút xác nhận ghi thành công (Quorum), client sẽ nhận được phản hồi thành công.
6.  **Hậu xử lý:** Nếu có nút nào bị ngoại tuyến, các nút còn lại sẽ lưu thông báo (hinted handoff) hoặc sử dụng tiến trình chạy nền (`repair`) để đồng bộ lại khi nút đó trực tuyến trở lại.

### Tổng kết
**Garage** là một ví dụ điển hình về việc sử dụng **Rust** để xây dựng hạ tầng đám mây. Nó không cố gắng bắt chước các hệ thống khổng lồ như Ceph hay Minio, mà tập trung vào sự **tinh gọn, dễ vận hành (Self-contained)** và **khả năng chống chịu cực cao** cho các cá nhân hoặc tổ chức muốn tự chạy Cloud trên hạ tầng phần cứng không chuyên dụng (Commodity Hardware) trải rộng nhiều nơi.