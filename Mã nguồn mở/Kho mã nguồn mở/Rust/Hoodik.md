Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Hoodik**, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và cách thức vận hành của hệ thống lưu trữ đám mây mã hóa đầu cuối (E2EE) này.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

Hoodik được xây dựng trên một ngăn xếp công nghệ (stack) hiện đại, tập trung vào hiệu suất và tính an toàn tuyệt đối:

*   **Ngôn ngữ lập trình:** 
    *   **Backend:** Rust (Sử dụng Actix-web). Lựa chọn này mang lại hiệu suất cực cao, khả năng xử lý đồng thời tốt và bộ nhớ an toàn (memory safety).
    *   **Frontend:** Vue 3, TypeScript và Tailwind CSS.
*   **Hệ thống Mẫu mã hóa (Cryptography):** 
    *   **Asymmetric (Bất đối xứng):** RSA-2048 dùng để bảo vệ khóa đối xứng và định danh người dùng.
    *   **Symmetric (Đối xứng):** AEGIS-128L (mặc định) và ChaCha20-Poly1305. Đáng chú ý là việc sử dụng **AEGIS-128L**, một thuật toán mã hóa cực nhanh nhờ tận dụng tập lệnh SIMD trên các CPU hiện đại và WASM.
    *   **WASM (WebAssembly):** Toàn bộ logic mã hóa nặng được đóng gói trong crate `transfer` và `cryptfns`, sau đó biên dịch sang WASM để chạy trong trình duyệt. Điều này đảm bảo tốc độ mã hóa gần tương đương phần cứng ngay trên web.
*   **Cơ sở dữ liệu & ORM:** 
    *   Sử dụng **Sea-ORM**, một thư viện ORM bất đồng bộ mạnh mẽ cho Rust.
    *   Hỗ trợ linh hoạt cả **SQLite** (cho nhu cầu nhỏ, tự lưu trữ) và **PostgreSQL** (cho quy mô lớn).
*   **Lưu trữ (Storage):** 
    *   Hỗ trợ đa nền tảng từ Local Filesystem đến **S3-compatible storage** (AWS S3, MinIO, Backblaze B2).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Hoodik tuân thủ nghiêm ngặt triết lý **"Trust No One" (Không tin tưởng bất kỳ ai)**:

*   **Kiến trúc Modular Monolith:** Dự án được chia thành nhiều "crates" (thư mục con như `auth`, `storage`, `cryptfns`, `admin`). Cách tiếp cận này giúp mã nguồn rõ ràng, dễ bảo trì nhưng vẫn đóng gói thành một file thực thi duy nhất (Single Binary), rất thuận tiện cho việc triển khai Docker.
*   **Client-Side Everything:** Server chỉ đóng vai trò là một "kho chứa mù". Logic mã hóa, giải mã, băm (hashing) để tìm kiếm đều thực hiện tại trình duyệt. Khóa bí mật của người dùng (Private Key) được mã hóa bằng mật khẩu (passphrase) trước khi gửi lên server.
*   **Zero-Knowledge Search:** Đây là một điểm cực kỳ thông minh. Hoodik không lưu tên tệp tin ở dạng văn bản rõ. Thay vào đó, nó chia nhỏ tên tệp thành các "token", băm chúng và lưu trữ. Khi người dùng tìm kiếm, trình duyệt sẽ băm từ khóa tìm kiếm và gửi lên server để so khớp các mã băm này.
*   **Fragment-based Key Sharing:** Khi chia sẻ liên kết, khóa giải mã được đặt sau dấu `#` trong URL (URL fragment). Theo tiêu chuẩn HTTP, phần này không bao giờ được gửi lên server, đảm bảo server không bao giờ biết khóa của liên kết công khai.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **WASM Integration:** Kỹ thuật dùng chung mã nguồn Rust cho cả Backend và Frontend (thông qua WASM) giúp giảm thiểu sai lệch trong logic mã hóa giữa hai đầu.
*   **Chunked & Concurrent Transfer:** Tệp tin được chia nhỏ thành các phần (chunks). Việc này cho phép:
    1. Tải lên đồng thời nhiều phần để tăng tốc độ.
    2. Khả năng tạm dừng và tiếp tục (Resume) dễ dàng.
    3. Kiểm tra tính toàn vẹn (Integrity) của từng phần nhỏ thông qua checksum (CRC).
*   **Blind Indexing:** Cách xử lý trong crate `admin` và `storage` cho thấy việc sử dụng các chỉ mục ẩn để thống kê dung lượng và loại tệp mà không cần biết nội dung bên trong.
*   **Tfa (Two-Factor Authentication):** Tích hợp sẵn TOTP mạnh mẽ ngay trong lõi hệ thống xác thực.

---

### 4. Luồng Hoạt động Hệ thống (System Operational Flow)

#### A. Luồng Đăng ký & Đăng nhập
1.  **Client:** Tạo cặp khóa RSA-2048.
2.  **Client:** Mã hóa Private Key bằng mật khẩu người dùng.
3.  **Client -> Server:** Gửi Public Key và Private Key đã mã hóa.
4.  **Login:** Server gửi Private Key đã mã hóa về, Client dùng mật khẩu giải mã cục bộ để lấy khóa gốc sử dụng cho các phiên làm việc.

#### B. Luồng Tải lên tệp tin (Upload)
1.  **Client:** Tạo một khóa đối xứng ngẫu nhiên (File Key) cho tệp.
2.  **Client:** Dùng WASM mã hóa tệp tin theo từng khối (chunk) bằng File Key.
3.  **Client:** Dùng Public Key của người dùng mã hóa chính cái File Key đó.
4.  **Client -> Server:** Gửi các khối tệp đã mã hóa + File Key đã mã hóa RSA.
5.  **Server:** Lưu các khối vào thư mục dữ liệu hoặc S3.

#### C. Luồng Chia sẻ (Sharing)
1.  **Client:** Tạo một Link Key ngẫu nhiên.
2.  **Client:** Mã hóa metadata và File Key bằng Link Key này.
3.  **Client:** Tạo URL có dạng `https://.../links/{id}#{Link_Key}`.
4.  **Recipient:** Trình duyệt của người nhận lấy `Link_Key` từ fragment URL để giải mã metadata và File Key, sau đó mới tải và giải mã tệp.

### Tổng kết
**Hoodik** là một dự án mẫu mực về cách kết hợp giữa hiệu suất của **Rust** và tính bảo mật của mã hóa đầu cuối. Hệ thống không chỉ giải quyết bài toán lưu trữ mà còn tối ưu hóa trải nghiệm người dùng thông qua việc sử dụng **WASM** và các kỹ thuật băm dữ liệu thông minh để giữ cho dữ liệu luôn ở trạng thái "Zero-Knowledge" đối với máy chủ.