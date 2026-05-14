Dựa trên các tệp tin và cấu trúc thư mục của dự án **HashiCorp Vault**, dưới đây là phân tích chuyên sâu về công nghệ cốt lõi, kiến trúc và các luồng hoạt động hệ thống.

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Vault là một hệ thống phân tán được xây dựng để đảm bảo tính bảo mật và tính sẵn sàng cao:

*   **Ngôn ngữ lập trình:** **Go (Golang)** là ngôn ngữ chủ đạo (chiếm ~68%). Go được chọn vì khả năng xử lý đồng thời (concurrency) tốt, hiệu suất cao và khả năng biên dịch thành tệp thực thi duy nhất, phù hợp cho môi trường bảo mật.
*   **Ngôn ngữ cấu hình:** **HCL (HashiCorp Configuration Language)**. Đây là ngôn ngữ đặc trưng của HashiCorp, giúp con người dễ đọc nhưng máy tính dễ phân tích.
*   **Giao tiếp & Serialization:** **Protobuf (Protocol Buffers)** và **gRPC**. Vault sử dụng Protobuf (thông qua công cụ `buf`) để định nghĩa các contract dữ liệu cho việc lưu trữ nội bộ và giao tiếp giữa các node hoặc plugin.
*   **Frontend:** Sử dụng **Ember.js** kết hợp với **TypeScript** (chiếm ~22%).
*   **Lưu trữ vật lý (Physical Layer):** Vault không tự quản lý dữ liệu thô mà dựa vào các backend như **Raft** (giao thức đồng thuận tích hợp sẵn), **Consul**, **PostgreSQL**, **S3**, hoặc **Azure**.
*   **Cơ chế mã hóa:** Sử dụng **AES-GCM 256-bit** để bảo vệ dữ liệu trước khi ghi xuống đĩa (mô hình Barrier).

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của Vault tuân theo nguyên tắc **"Barrier-Based Architecture"**:

*   **Nguyên tắc "Mọi thứ là một đường dẫn" (Everything is a Path):** Tương tự như triết lý của UNIX, Vault coi mọi bí mật, cấu hình xác thực và chính sách là một đường dẫn API (ví dụ: `secret/data/my-password`).
*   **Tách biệt giữa Logical và Physical:**
    *   **Physical Store:** Là nơi lưu trữ dữ liệu đã mã hóa. Tầng này không tin cậy (untrusted).
    *   **Logical Layer:** Xử lý logic nghiệp vụ, kiểm tra quyền (ACL) và quản lý vòng đời bí mật.
*   **Mô hình Plugin:** Vault cực kỳ linh hoạt. Các phương thức xác thực (Auth Methods), công cụ tạo bí mật (Secret Engines) và kiểm tra logs (Audit Devices) đều được thiết kế như các plugin (có thể là builtin hoặc tệp thực thi bên ngoài).
*   **Triết lý "Zero Trust":** Vault giả định rằng kẻ tấn công có thể truy cập được đĩa cứng vật lý. Dữ liệu chỉ có thể giải mã khi Vault ở trạng thái "Unsealed" (đã mở khóa).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Shamir's Secret Sharing:** Khi khởi tạo, Vault tạo ra một khóa master nhưng không lưu trữ nó. Thay vào đó, khóa này được chia thành nhiều phần (key shares). Để mở khóa Vault (Unseal), cần một số lượng phần tối thiểu (threshold).
*   **mlock (Memory Locking):** Vault sử dụng syscall `mlock` trong Go để ngăn hệ điều hành chuyển dữ liệu nhạy cảm từ RAM sang Swap trên đĩa cứng, nhằm tránh việc rò rỉ bí mật qua bộ nhớ ảo.
*   **Leasing & TTL:** Mọi bí mật do Vault cấp phát đều đi kèm với một "Lease" (hợp đồng thuê). Khi hết hạn TTL (Time To Live), Vault tự động thu hồi bí mật đó. Đây là kỹ thuật then chốt để quản lý các "Dynamic Secrets" (bí mật động).
*   **High Availability (HA) với Raft:** Vault tích hợp giao thức đồng thuận Raft (`physical/raft/`), cho phép cụm Vault tự bầu chọn Leader và sao chép trạng thái mà không cần phụ thuộc vào bên thứ ba như Consul.
*   **JSON Merge Patch:** Hỗ trợ cập nhật từng phần cho dữ liệu KV thông qua chuẩn RFC 7396, giúp tối ưu hóa việc sửa đổi các cấu hình lớn.

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng Khởi động và Mở khóa (Initialization & Unsealing)
1.  Vault khởi động ở trạng thái **Sealed**. Nó có quyền truy cập vào storage nhưng không thể giải mã dữ liệu.
2.  Người dùng cung cấp đủ số lượng khóa (Key Shares).
3.  Vault tái cấu trúc Khóa Master từ các shares, sau đó giải mã Khóa Barrier (khóa thực sự dùng để mã hóa dữ liệu trên đĩa).
4.  Vault chuyển sang trạng thái **Active** và bắt đầu phục vụ yêu cầu.

#### B. Luồng Xác thực và Truy cập Bí mật (Auth & Access)
1.  **Xác thực:** Client gửi thông tin định danh (Token, AppRole, LDAP, v.v.) đến một `auth/` path.
2.  **Cấp Token:** Nếu thành công, Vault trả về một tệp tin Client Token gắn liền với các chính sách (Policies).
3.  **Yêu cầu dữ liệu:** Client dùng Token đó gửi yêu cầu đến `logical/` path (ví dụ: `database/creds/readonly`).
4.  **Kiểm tra ACL:** Vault kiểm tra xem Token có quyền thực hiện hành động trên path đó không.
5.  **Tạo bí mật động:** Nếu là Secret Engine (như Database), Vault sẽ kết nối với DB thật, tạo một user tạm thời và trả về cho Client kèm theo một `lease_id`.

#### C. Luồng Quản lý Vòng đời (Renewal & Revocation)
1.  **Lifetime Watcher:** Tệp `api/lifetime_watcher.go` cho thấy client có thể chạy một tiến trình ngầm để theo dõi thời hạn của bí mật.
2.  **Gia hạn:** Nếu được phép, client yêu cầu `Renew`. Vault cập nhật lại thời hạn trên storage.
3.  **Thu hồi:** Nếu hết hạn hoặc bị thu hồi thủ công, Vault sẽ chạy logic rollback (ví dụ: xóa user trong Database) để đảm bảo bí mật không còn hiệu lực.

### Tổng kết
Vault là một hệ thống được thiết kế với độ an toàn cực cao, sử dụng các kỹ thuật mật mã học cổ điển (Shamir) kết hợp với các kỹ thuật hệ thống hiện đại (Go, Raft). Cấu trúc mã nguồn của nó thể hiện sự phân tách rất rõ ràng giữa giao thức (API), logic (Vault Core) và lưu trữ (Physical), tạo nên một tiêu chuẩn công nghiệp cho việc quản lý bí mật.