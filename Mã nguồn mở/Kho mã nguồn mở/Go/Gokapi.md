Chào bạn, đây là bản phân tích chi tiết về kho lưu trữ **Gokapi** dựa trên mã nguồn và tài liệu bạn cung cấp.

---

### 1. Công nghệ Cốt lõi (Core Technologies)

Gokapi được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào hiệu suất và khả năng đóng gói (portability):

*   **Ngôn ngữ lập trình:** **Go (Golang)** phiên bản mới nhất (v1.24/1.25). Việc dùng Go giúp tạo ra các file thực thi (binary) tĩnh, không phụ thuộc thư viện hệ thống, rất dễ triển khai.
*   **WebAssembly (WASM):** Đây là điểm đặc biệt. Gokapi sử dụng Go để biên dịch sang WASM (`cmd/wasmdownloader` và `cmd/wasme2e`), cho phép thực hiện việc **mã hóa và giải mã ngay trên trình duyệt** người dùng.
*   **Cơ sở dữ liệu:**
    *   **SQLite (modernc.org/sqlite):** Phiên bản CGO-free, giúp việc biên dịch chéo (cross-compile) sang Windows/Arm64 dễ dàng.
    *   **Redis:** Hỗ trợ lưu trữ phân tán và tốc độ cao cho các hệ thống lớn.
*   **Lưu trữ đám mây:** Tích hợp **AWS S3 SDK**, hỗ trợ các dịch vụ tương thích S3 như Backblaze B2, Cloudflare R2.
*   **Mã hóa:** Sử dụng thư viện `secure-io/sio-go` để mã hóa luồng (streaming encryption) theo chuẩn DARE (Data-At-Rest-Encryption).
*   **Xác thực:** Hỗ trợ **OpenID Connect (OIDC)** để tích hợp với các hệ thống Identity Provider (IdP) như Authelia, Keycloak hoặc Google.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Gokapi thể hiện tư duy của một hệ thống Cloud-native nhưng vẫn cực kỳ nhẹ nhàng:

*   **Tính trừu tượng hóa (Abstraction):**
    *   **Storage Interface:** Gokapi định nghĩa các interface cho việc lưu trữ. Code xử lý logic không cần biết file đang nằm ở ổ cứng cục bộ hay trên S3.
    *   **Database Abstraction:** Hệ thống có một lớp trung gian (`dbabstraction`) để chuyển đổi linh hoạt giữa SQLite và Redis mà không làm thay đổi logic nghiệp vụ.
*   **Mã hóa đa tầng (Layered Encryption):** Gokapi cung cấp 4 cấp độ từ không mã hóa đến **End-to-End Encryption (E2E)**. Tư duy ở đây là đặt quyền riêng tư lên hàng đầu: ở chế độ E2E, ngay cả quản trị viên server cũng không thể xem nội dung file vì key nằm ở trình duyệt người dùng.
*   **Xử lý File lớn bằng Chunking:** Thay vì tải toàn bộ file vào RAM (gây crash trên các server yếu như Raspberry Pi), Gokapi chia file thành các phần nhỏ (mặc định 45MB). Các chunk này được xử lý theo luồng (streaming), giúp tối ưu bộ nhớ.
*   **Chống trùng lặp (Deduplication):** Hệ thống tính toán Hash của file. Nếu hai người dùng tải lên cùng một file, server chỉ lưu một bản copy duy nhất để tiết kiệm không gian.

---

### 3. Các kỹ thuật chính nổi bật (Technical Highlights)

1.  **Sử dụng Go trong WebAssembly:** Thay vì viết lại logic mã hóa bằng JavaScript (có thể chậm hoặc tiềm ẩn lỗi bảo mật), tác giả dùng Go biên dịch sang WASM. Điều này đảm bảo **logic mã hóa ở Client và Server là đồng nhất**.
2.  **CGO-Free:** Việc sử dụng các thư viện thuần Go (như SQLite thuần Go) giúp Gokapi có thể chạy trên gần như mọi kiến trúc CPU (x86, ARM) mà không cần cài đặt các bộ thư viện C phức tạp.
3.  **Hotlinking thông minh:** Gokapi hỗ trợ tạo link trực tiếp (hotlink) cho ảnh. Kỹ thuật này yêu cầu xử lý các HTTP Header (`Content-Type`, `Cache-Control`) rất kỹ để tránh trình duyệt tải lại file không cần thiết.
4.  **Graceful Shutdown:** Trong `cmd/gokapi/Main.go`, hệ thống lắng nghe các tín hiệu hệ điều hành (SIGTERM, SIGINT) để đóng cơ sở dữ liệu và dừng server một cách an toàn, tránh mất mát dữ liệu.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của Gokapi có thể tóm tắt như sau:

1.  **Thiết lập (Setup):** Khi chạy lần đầu, Gokapi mở một server web tạm thời để người dùng cấu hình DB, URL và phương thức xác thực qua giao diện trực quan.
2.  **Tải lên (Upload):**
    *   Client chia nhỏ file thành các chunk.
    *   Nếu bật E2E, WASM sẽ mã hóa chunk trước khi gửi.
    *   Server nhận chunk, kiểm tra Hash để xem file đã tồn tại chưa (Deduplication).
    *   Lưu vào Local Storage hoặc đẩy lên S3.
3.  **Quản lý (Management):** Server lưu Metadata (tên file, thời gian hết hạn, số lần tải cho phép) vào SQLite/Redis.
4.  **Tải về (Download):**
    *   Người dùng truy cập link. Server kiểm tra điều kiện (mật khẩu, số lần tải, thời gian).
    *   Nếu file mã hóa, WASM trên trình duyệt sẽ nhận luồng dữ liệu và giải mã trực tiếp (Stream decryption) để người dùng lưu file.
5.  **Dọn dẹp (Cleanup):** Một worker chạy ngầm định kỳ quét cơ sở dữ liệu. File nào hết hạn hoặc đạt giới hạn lượt tải sẽ bị xóa vĩnh viễn khỏi bộ lưu trữ.

---
**Đánh giá chung:** Gokapi là một dự án có kỹ thuật triển khai rất sạch sẽ, mã nguồn tổ chức khoa học (đặc biệt là phần phân tách giữa `cmd` và `internal`). Việc kết hợp Go và WASM cho thấy một tư duy hiện đại về bảo mật và hiệu suất ứng dụng web.