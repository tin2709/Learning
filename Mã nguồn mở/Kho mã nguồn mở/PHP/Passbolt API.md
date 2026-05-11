Dựa trên cấu trúc thư mục và mã nguồn của **Passbolt API** (phiên bản 5.11.0), đây là một hệ thống quản lý mật khẩu cấp doanh nghiệp được xây dựng trên triết lý bảo mật tuyệt đối và khả năng mở rộng mô-đun. Dưới đây là phân tích chi tiết:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ & Framework:** Sử dụng **PHP 8.2+** cùng với **CakePHP 5**. Đây là một lựa chọn ổn định, cung cấp cấu trúc MVC chặt chẽ và các lớp bảo mật tích hợp sẵn (ORM, CSRF protection, v.v.).
*   **Mã hóa (Trọng tâm):** Hệ thống dựa trên **GnuPG (OpenPGP)**. Sử dụng `ext-gnupg` cho các thao tác phía server và `openpgp-php` để xử lý dữ liệu. Toàn bộ bí mật được lưu trữ dưới dạng khối văn bản đã được bọc thép (armored blocks), đảm bảo server không bao giờ thấy mật khẩu ở dạng thô (Zero-knowledge).
*   **Xác thực:** Kết hợp nhiều phương thức: GPG Authentication (Challenge-Response), JWT (`firebase/php-jwt`) cho các phiên làm việc di động, và hỗ trợ MFA đa dạng (TOTP, Yubikey, Duo Security).
*   **Cơ sở dữ liệu:** Sử dụng MySQL/MariaDB với hệ thống Migrations cực kỳ chi tiết, cho thấy lịch sử phát triển lâu dài và quản lý schema chặt chẽ.
*   **Kiểm soát chất lượng:** Dự án áp dụng các tiêu chuẩn kiểm thử khắt khe với **PHPUnit**, phân tích tĩnh bằng **PHPStan (Level 6)** và **Psalm (Level 4)**, đảm bảo code sạch và ít lỗi logic.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Plugin-Driven Architecture:** Passbolt không xây dựng một khối monolith cồng kềnh. Thay vào đó, các tính năng như `MFA`, `Folders`, `SecretRevisions`, `Import/Export` được tách thành các plugin riêng biệt trong thư mục `plugins/PassboltCe/`. Điều này cho phép dễ dàng bảo trì và nâng cấp các phiên bản Community (CE) hoặc Professional (Pro) một cách độc lập.
*   **Service-Oriented Logic:** Các logic nghiệp vụ phức tạp không nằm trong Controller mà được đẩy vào tầng **Service** (ví dụ: `src/Service/Secrets/SecretsCreateService.php`). Controller chỉ đóng vai trò điều phối yêu cầu và phản hồi.
*   **Kiến trúc E2EE (End-to-End Encryption):** Thiết kế database và API xoay quanh việc lưu trữ các khóa công khai (`gpgkeys`) và các bí mật đã mã hóa (`secrets`). Quyền truy cập được quản lý thông qua việc chia sẻ các khóa phiên đã mã hóa giữa các người dùng.
*   **Auditing & Traceability:** Hệ thống có các bảng history cho mọi thực thể (`entities_history`, `secrets_history`). Mọi hành động của người dùng đều được ghi lại trong `action_logs` phục vụ mục đích tuân thủ bảo mật.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Hệ thống Healthcheck chuyên sâu:** Tầng `src/Service/Healthcheck/` chứa hàng chục lớp kiểm tra từ cấu hình server, quyền ghi thư mục, đồng bộ thời gian đến tính hợp lệ của khóa GPG. Đây là kỹ thuật giúp giảm thiểu lỗi vận hành trên các môi trường tự triển khai (on-premise).
*   **Email Redactor Pattern:** Trong tầng thông báo, Passbolt sử dụng các `Redactor` (ví dụ: `ResourceUpdateEmailRedactor`). Kỹ thuật này giúp tùy biến nội dung email một cách linh hoạt theo từng sự kiện mà không làm rối mã nguồn chính của dịch vụ gửi thư.
*   **UUID-based Identification:** Sử dụng UUID thay vì ID tăng tự động cho mọi bản ghi, giúp tăng tính bảo mật (không thể đoán định ID) và hỗ trợ tốt cho việc đồng bộ hóa dữ liệu giữa các node.
*   **GPG Auth Headers:** Một Middleware tùy chỉnh (`GpgAuthHeadersMiddleware`) được sử dụng để xử lý việc xác thực dựa trên chữ ký số trong các header HTTP, một kỹ thuật xác thực mạnh mẽ hơn nhiều so với login truyền thống.

### 4. Luồng Hoạt động Hệ thống (System Operation Flow)

1.  **Thiết lập (Setup):** Người dùng tạo cặp khóa GPG trên trình duyệt. Khóa công khai được gửi lên API thông qua `UsersRegisterController` và lưu vào bảng `gpgkeys`.
2.  **Tạo bí mật (Create Secret):**
    *   Trình duyệt mã hóa mật khẩu bằng khóa công khai của server (và khóa của chính người dùng).
    *   API nhận dữ liệu đã mã hóa, `SecretsCreateService` lưu vào bảng `secrets`.
    *   Hệ thống ghi log vào `action_logs`.
3.  **Chia sẻ (Share):**
    *   Khi User A chia sẻ cho User B, trình duyệt của A lấy khóa công khai của B, giải mã mật khẩu bằng khóa riêng của A, sau đó mã hóa lại bằng khóa công khai của B.
    *   API nhận "bản sao" đã mã hóa mới này và lưu vào bảng `permissions` và `secrets`.
4.  **Truy vấn (Read):** Người dùng gửi yêu cầu. API kiểm tra quyền truy cập trong bảng `permissions`. Nếu hợp lệ, trả về khối GPG đã mã hóa. Trình duyệt người dùng sẽ dùng khóa riêng (private key) lưu cục bộ để giải mã.
5.  **Kiểm tra trạng thái (Healthcheck):** Hệ thống thường xuyên chạy các lệnh CLI hoặc endpoint để đảm bảo môi trường GPG và Database luôn ở trạng thái an toàn.

### Tổng kết
Passbolt API là một dự án PHP mẫu mực về **Security-first**. Mã nguồn thể hiện sự kết hợp hoàn hảo giữa một framework hiện đại (CakePHP 5) với các tiêu chuẩn bảo mật mã nguồn mở (OpenPGP), tạo ra một hệ thống vừa linh hoạt trong phát triển vừa cực kỳ an toàn trong vận hành.