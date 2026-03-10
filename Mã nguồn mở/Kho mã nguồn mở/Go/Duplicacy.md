Dựa trên các tệp tin mã nguồn của dự án **Duplicacy**, tôi xin gửi tới bạn bản phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của công cụ này:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Duplicacy được xây dựng chủ yếu bằng ngôn ngữ **Go (Golang)**, tận dụng khả năng xử lý song song mạnh mẽ và tính đa nền tảng:

*   **Lưu trữ đám mây đa dạng:** Hỗ trợ hầu hết các SDK lớn như `aws-sdk-go` (S3), `azure-sdk-for-go`, `google-api-go-client` (Drive/GCS), và các giao thức tiêu chuẩn như SFTP, WebDAV, SMB.
*   **Thuật toán băm (Hashing):** Sử dụng **Blake2b** (thông qua `minio/blake2b-simd`) cho tốc độ băm cực nhanh trên các CPU hiện đại, bên cạnh SHA256 truyền thống.
*   **Nén dữ liệu:** Hỗ trợ nhiều cấp độ nén gồm **LZ4** (nhanh), **Zstd** (hiệu quả cao), và **Zlib**.
*   **Mã hóa (Encryption):**
    *   **Symmetric:** AES-256 trong chế độ GCM.
    *   **Asymmetric:** RSA-256 (RSA-OAEP) để bảo vệ khóa AES, cho phép sao lưu mà không cần giữ mật khẩu giải mã trên máy khách.
*   **Bảo vệ dữ liệu:** Sử dụng **Reed-Solomon Erasure Coding** (`klauspost/reedsolomon`) để phục hồi dữ liệu ngay cả khi một số chunk bị hỏng hoặc mất.
*   **Tương tác hệ thống:** Hỗ trợ **VSS (Volume Shadow Copy)** trên Windows và **APFS Snapshots** trên macOS để sao lưu các tệp đang mở.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Điểm khác biệt lớn nhất của Duplicacy nằm ở hai triết lý thiết kế:

*   **Database-less (Không cơ sở dữ liệu):** Khác với các công cụ như Restic hay Duplicati cần một DB cục bộ để quản lý các "chunk" (mảnh dữ liệu), Duplicacy coi chính cấu trúc thư mục trên Cloud là index. Tên tệp tin chính là mã băm (hash) của nội dung chunk đó. Điều này giúp hệ thống cực kỳ nhẹ, không sợ hỏng DB cục bộ và dễ dàng khôi phục từ bất kỳ đâu.
*   **Lock-free Deduplication (Chống trùng lặp không khóa):** Đây là sáng kiến cốt lõi được công bố trong bài báo IEEE của tác giả. Nó cho phép nhiều máy tính khác nhau cùng sao lưu vào một tài khoản Cloud đồng thời mà không cần giao tiếp với nhau và cũng không cần cơ chế khóa (locking) phức tạp, trong khi vẫn đạt được hiệu quả chống trùng lặp dữ liệu (cross-computer deduplication).

### 3. Các kỹ thuật chính (Key Techniques)
*   **Variable-size Chunking (Cắt nhỏ dữ liệu kích thước thay đổi):** Sử dụng thuật toán **Buzhash** (rolling hash) để xác định điểm cắt dữ liệu. Kỹ thuật này giúp xử lý tốt trường hợp thêm/bớt dữ liệu ở giữa tệp, vì nó tìm lại được các đoạn cũ, trong khi kỹ thuật cắt cố định (fixed-size) sẽ làm hỏng toàn bộ logic khớp dữ liệu sau vị trí thay đổi.
*   **Nesting Levels (Phân cấp thư mục):** Để tránh việc có hàng triệu tệp trong một thư mục đơn lẻ (gây chậm cho nhiều hệ thống tệp), Duplicacy chia chunk vào các thư mục con dựa trên 2 chữ cái đầu của mã băm (ví dụ: `chunks/4d/53...`).
*   **Fossilization (Hóa thạch):** Để thực hiện việc xóa dữ liệu cũ (Prune) mà không cần khóa, Duplicacy không xóa trực tiếp. Nó đổi tên chunk cần xóa thành tệp "fossil" (`.fsl`). Nếu sau một khoảng thời gian chờ (grace period) mà không có máy tính nào cần đến chunk đó, nó mới chính thức bị xóa.
*   **Snapshot Migration:** Khả năng sao chép hoặc di chuyển toàn bộ các bản sao lưu (snapshots) giữa các vùng lưu trữ khác nhau (ví dụ từ S3 sang B2) một cách nguyên vẹn.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

#### Quy trình Backup:
1.  **Quét (Indexing):** Hệ thống quét các tệp cục bộ, kiểm tra thời gian sửa đổi (mtime) để xác định tệp thay đổi.
2.  **Cắt mảnh (Chunking):** Tệp được chạy qua thuật toán Buzhash để chia thành các chunk nhỏ (trung bình 4MB).
3.  **Băm & Kiểm tra:** Tính mã băm của mỗi chunk. Kiểm tra trên Cloud xem tệp có tên là mã băm đó đã tồn tại chưa.
4.  **Xử lý Chunk:** Nếu chưa có, chunk sẽ được nén -> mã hóa -> (tùy chọn) Erasure Coding -> tải lên Cloud.
5.  **Snapshot:** Sau khi tất cả chunk đã tải lên, một tệp "Snapshot" chứa danh sách các mã băm của phiên bản đó sẽ được tải lên thư mục `snapshots/`.

#### Quy trình Prune (Xóa bản cũ):
1.  **Đánh dấu:** Liệt kê tất cả các chunk được tham chiếu bởi các bản sao lưu cần giữ lại.
2.  **Hóa thạch:** Các chunk không được tham chiếu sẽ bị đổi tên thành `.fsl`.
3.  **Xóa thực sự:** Ở lần chạy Prune tiếp theo, các tệp `.fsl` cũ hơn thời gian quy định sẽ bị xóa bỏ hoàn toàn.

### Tổng kết
Duplicacy là một ví dụ điển hình về việc áp dụng nghiên cứu học thuật vào phần mềm thực tế. Nó giải quyết bài toán khó nhất của sao lưu đám mây là **tính nhất quán khi truy cập đồng thời** bằng một kiến trúc tối giản, không dựa vào DB và tận dụng tối đa sức mạnh của mã băm nội dung.