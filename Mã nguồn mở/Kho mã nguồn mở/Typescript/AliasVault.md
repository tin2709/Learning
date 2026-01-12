Dưới đây là phân tích chi tiết về dự án **AliasVault** dựa trên cấu trúc thư mục và mã nguồn bạn đã cung cấp:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

AliasVault là một ứng dụng Full-stack hiện đại, đa nền tảng với sự kết hợp của nhiều công nghệ mạnh mẽ:

*   **Backend (.NET Ecosystem):** Sử dụng **.NET (C#)** cho hệ thống API, quản trị (Admin) và các dịch vụ chạy ngầm. 
    *   **Entity Framework Core:** Quản lý database (PostgreSQL cho Server, SQLite cho Client).
    *   **SmtpService:** Tự xây dựng Server SMTP riêng để xử lý email alias.
*   **Frontend (Web & UI):** 
    *   **React:** Sử dụng cho Browser Extension và ứng dụng Web chính.
    *   **Blazor (WASM):** Được sử dụng trong phần `AliasVault.Client` (Web App), cho phép chạy mã C# trực tiếp trên trình duyệt.
    *   **Tailwind CSS:** Framework CSS chính để xây dựng giao diện nhất quán.
*   **Browser Extension:** 
    *   **WXT Framework:** Một công cụ hiện đại để phát triển Web Extension giúp hỗ trợ nhiều trình duyệt (Chrome, Firefox, Edge, Safari).
*   **Mobile App:** 
    *   **React Native / Expo:** Dùng cho ứng dụng iOS/Android.
    *   **Kotlin (Android) & Swift (iOS):** Các module Native được viết riêng để xử lý Autofill và bảo mật sâu trong hệ điều hành (như Biometric, Keystore).
*   **Database:**
    *   **PostgreSQL:** Lưu trữ dữ liệu phía Server (đã được mã hóa).
    *   **SQLite (Wasm):** Sử dụng trực tiếp trong trình duyệt/extension để lưu trữ Vault tạm thời dưới dạng file db mã hóa.

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)

Dự án tuân thủ triết lý **Zero-Knowledge Architecture** (Kiến trúc không tri thức):

*   **Client-Side Everything:** Mọi hoạt động mã hóa và giải mã đều diễn ra ở thiết bị người dùng. Master Password không bao giờ được gửi lên Server.
*   **Key Derivation:** Sử dụng **Argon2id** – thuật toán chống brute-force mạnh nhất hiện nay – để biến mật khẩu chính thành khóa mã hóa (Encryption Key).
*   **Symmetry & Asymmetry Hybrid:** 
    *   Dùng **AES-256-GCM** để mã hóa kho dữ liệu (Vault).
    *   Dùng **RSA-OAEP** để mã hóa email. Server nhận email, dùng Public Key của người dùng mã hóa lại rồi mới lưu. Chỉ Private Key của người dùng (nằm trong Vault đã giải mã) mới đọc được email.
*   **SRP (Secure Remote Password) Protocol:** Một giao thức xác thực cực kỳ bảo mật, cho phép Client chứng minh với Server rằng mình biết mật khẩu mà không cần gửi mật khẩu đó qua mạng.
*   **Shared Models:** Có một thư mục `shared/` chứa logic dùng chung cho cả Web, Extension và Mobile (như bộ tạo danh tính, bộ tạo mật khẩu, các model SQL). Điều này đảm bảo tính nhất quán (Consistency) cao.

### 3. Các kỹ thuật chính nổi bật (Technical Highlights)

*   **Built-in Email Server:** Khác với các trình quản lý mật khẩu khác, AliasVault tích hợp sẵn server SMTP. Kỹ thuật này cho phép tạo các "email ảo" (alias) ngay lập tức và bảo vệ danh tính thực của người dùng.
*   **WebAuthn Interceptor:** Extension có kỹ thuật "đánh chặn" (intercept) các yêu cầu WebAuthn/Passkey của trình duyệt để thay thế bằng trình quản lý của chính nó, giúp người dùng đăng nhập không cần mật khẩu một cách liền mạch.
*   **Mobile-Unlock (QR-Based):** Sử dụng cặp khóa RSA để cho phép ứng dụng di động "mở khóa" cho trình duyệt thông qua việc quét mã QR, truyền Encryption Key một cách an toàn qua trung gian Server mà Server vẫn không thể đọc được khóa này.
*   **Database Syncing:** Kỹ thuật đồng bộ hóa dựa trên `vaultRevisionNumber`. Extension so sánh phiên bản cục bộ với Server để quyết định tải về hay đẩy dữ liệu lên, đảm bảo không mất dữ liệu khi dùng nhiều thiết bị.
*   **Offscreen Document (Chrome MV3):** Sử dụng tài liệu ẩn (offscreen) để xử lý các tác vụ như xóa Clipboard tự động sau một khoảng thời gian, đáp ứng các tiêu chuẩn bảo mật khắt khe của Manifest V3.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Dựa trên mã nguồn, quy trình hoạt động cơ bản như sau:

1.  **Đăng ký/Đăng nhập:** Người dùng nhập Master Password -> App dùng **Argon2id** tạo ra khóa cục bộ -> Dùng **SRP** để xác thực với Server -> Nếu thành công, Server gửi Vault đã mã hóa (Blob) về thiết bị.
2.  **Mở khóa Vault:** Khóa cục bộ giải mã Blob SQLite -> Load dữ liệu vào bộ nhớ (Memory) -> Extension/App sẵn sàng sử dụng.
3.  **Tự động điền (Autofill):** Content Script trên trình duyệt phát hiện các trường input (Username/Password) -> So khớp domain với Vault -> Hiện biểu tượng AliasVault -> Người dùng click -> Điền dữ liệu và kích hoạt sự kiện `input/change` để đánh lừa script kiểm tra của website.
4.  **Xử lý Email Alias:** Một người gửi email đến alias của bạn -> **SmtpService** nhận mail -> Truy vấn Public Key của bạn từ DB -> Mã hóa nội dung mail -> Lưu vào DB. Khi bạn mở App -> App tải mail về -> Dùng Private Key trong Vault để giải mã và hiển thị.
5.  **Tạo Identity:** Khi cần tài khoản mới, App dùng `IdentityGenerator` (dựa trên từ điển ngôn ngữ trong `shared/`) để tạo ngẫu nhiên Họ tên, Ngày sinh, Địa chỉ hợp lý và một Email Alias mới.

**Kết luận:** AliasVault là một dự án có độ phức tạp kỹ thuật cao, đặc biệt chú trọng vào bảo mật lớp sâu và quyền riêng tư tuyệt đối thông qua việc tích hợp email alias và kiến trúc Zero-knowledge.