Dựa trên cấu trúc thư mục và nội dung mã nguồn của **Ory Kratos**, đây là phân tích chuyên sâu về hệ thống quản lý danh tính (Identity Management) chuẩn Cloud-native này:

### 1. Công nghệ Cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình:** Sử dụng **Go (Golang)** làm chủ đạo. Lựa chọn này giúp tận dụng khả năng xử lý đồng thời (concurrency) cực tốt cho các tác vụ như hashing mật khẩu, gửi mail/SMS qua worker và quản lý hàng nghìn kết nối API cùng lúc.
*   **Cơ sở dữ liệu:** Hỗ trợ đa dạng qua lớp trừu tượng (Persistence Layer): **PostgreSQL, MySQL, CockroachDB và SQLite**. Sử dụng thư viện `ory/pop` để quản lý Migration và truy vấn SQL.
*   **Thuật toán Hashing mật khẩu:** Mặc định sử dụng **Argon2** (thuật toán thắng cuộc trong Password Hashing Competition), ngoài ra còn hỗ trợ Bcrypt, PBKDF2 và Scrypt. Trong mã nguồn (`cmd/hashers/argon2`), Kratos có hẳn một bộ công cụ "calibrate" để tự động đo đạc cấu hình phần cứng và đề xuất thông số Argon2 tối ưu.
*   **Giao thức bảo mật & Auth:** Tích hợp sẵn **WebAuthn/FIDO2** (cho Passkeys), **OIDC** (cho Social Login), và **TOTP**.
*   **Jsonnet:** Sử dụng ngôn ngữ cấu hình Jsonnet để ánh xạ (map) dữ liệu từ các nhà cung cấp bên thứ ba (Google, Facebook...) vào Traits của người dùng một cách linh hoạt mà không cần thay đổi code Go.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Kratos là minh chứng cho triết lý **"API-first"** và **"Headless"**:

*   **Headless Architecture:** Kratos không đi kèm giao diện (UI). Nó chỉ cung cấp các HTTP API (Public & Admin). Điều này cho phép lập trình viên tự xây dựng UI bằng bất cứ công nghệ nào (React, Vue, Flutter) trong khi logic nghiệp vụ phức tạp về danh tính nằm trọn trong Kratos.
*   **Identity Schema (Trait-based):** Thay vì quy định cứng các cột `first_name`, `last_name`, Kratos sử dụng **JSON Schema**. Bạn định nghĩa cấu hình danh tính trong file JSON, Kratos sẽ tự động validate dữ liệu dựa trên schema đó. Đây là tư duy cực kỳ linh hoạt cho các hệ thống đa quốc gia hoặc đa sản phẩm.
*   **Self-Service Flows:** Mọi quy trình (Login, Registration, Recovery, Verification, Settings) đều được mô hình hóa thành một **"Flow"**. Mỗi Flow có vòng đời (Lifespan), trạng thái (State) và ID riêng, được lưu vào DB để duy trì tính liên tục của phiên làm việc (Continuity).
*   **Registry Pattern:** Trong gói `driver`, Kratos sử dụng một cấu trúc Registry trung tâm để quản lý Dependency Injection. Tất cả các thành phần (Persister, Cipher, Logger, Config) đều được đăng ký vào Registry này, giúp việc kiểm thử (Testing) và mở rộng cực kỳ dễ dàng.

### 3. Kỹ thuật Lập trình Đặc sắc (Distinctive Programming Techniques)

*   **Continuity Containers:** Gói `continuity` quản lý trạng thái của các quy trình bị gián đoạn (ví dụ: đang đăng ký thì phải đi xác thực email). Kratos lưu trạng thái này vào một "container" trong DB thay vì giữ trong bộ nhớ, giúp hệ thống không trạng thái (stateless) và dễ dàng mở rộng theo chiều ngang.
*   **Courier Worker:** Kratos không gửi Email/SMS trực tiếp trong luồng API chính. Thay vào đó, nó ghi tin nhắn vào DB. Một thành phần gọi là **Courier** (xem `courier/`) chạy như một worker riêng biệt, liên tục quét DB và gửi tin nhắn qua SMTP hoặc HTTP. Kỹ thuật này giúp API phản hồi cực nhanh và có khả năng retry khi việc gửi tin nhắn thất bại.
*   **Cipher Layer:** Trong thư mục `cipher/`, Kratos triển khai lớp mã hóa với các thuật toán **AES-GCM** và **XChaCha20-Poly1305**. Mọi dữ liệu nhạy cảm trước khi xuống DB đều được đi qua lớp này để đảm bảo dù DB bị lộ, dữ liệu vẫn an toàn.
*   **Snapshot Testing:** Dự án sử dụng rất nhiều kỹ thuật Snapshot Testing (thư mục `.snapshots`). Kỹ thuật này ghi lại kết quả phản hồi của API và so sánh với phiên bản cũ, đảm bảo các thay đổi code không làm hỏng cấu trúc dữ liệu trả về cho khách hàng.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

Ví dụ luồng **Đăng ký (Registration)**:

1.  **Initialize Flow:** Client gọi API Public để khởi tạo luồng đăng ký. Kratos tạo một `RegistrationFlow` record trong DB và trả về một `flow_id`.
2.  **Get Flow:** UI fetch thông tin flow bằng `flow_id`. Kratos trả về danh sách các "Nodes" (các input field cần hiển thị dựa trên JSON Schema).
3.  **Submit Flow:** User điền thông tin và submit về Kratos.
4.  **Validation:** Kratos validate dữ liệu dựa trên JSON Schema đã cấu hình.
5.  **Hooks Execution:** Sau khi đăng ký thành công, Kratos chạy các "Hooks" (ví dụ: tạo session ngay lập tức, gửi email xác thực, hoặc gọi một Webhook bên ngoài để đồng bộ dữ liệu sang hệ thống khác).
6.  **Completion:** Kratos trả về kết quả (thường là một Session Token hoặc Cookie).

### Tổng kết
Ory Kratos không chỉ là một phần mềm Auth, nó là một **Identity Engine**. Hệ thống được thiết kế để giải quyết bài toán "danh tính" ở quy mô lớn (billions of users) với độ tùy biến tuyệt đối thông qua JSON Schema và API, tách biệt hoàn toàn giữa logic bảo mật và trải nghiệm người dùng (UX).