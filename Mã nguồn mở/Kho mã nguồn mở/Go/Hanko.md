Dưới đây là bản phân tích chi tiết về dự án **Hanko** dựa trên mã nguồn và cấu trúc thư mục bạn đã cung cấp, được trình bày dưới dạng một tệp `README.md` chuyên sâu bằng tiếng Việt.

---

# Phân Tích Kỹ Thuật Dự Án Hanko (Authentication & User Management)

## 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án Hanko được xây dựng trên mô hình **Monorepo**, kết hợp giữa các ngôn ngữ hiệu suất cao và giao diện linh hoạt:

*   **Backend:** 
    *   **Ngôn ngữ:** Go (Golang) - Chiếm 66% mã nguồn, đảm bảo tính đồng thời và hiệu suất cao.
    *   **Web Framework:** `Echo` - Một framework nhẹ và mạnh mẽ cho Go.
    *   **Cơ sở dữ liệu:** Hỗ trợ PostgreSQL và MySQL (sử dụng `Pop/Buffalo` để quản lý Migration và ORM).
    *   **Xác thực:** 
        *   `WebAuthn (FIDO2/Passkeys)`: Sử dụng thư viện `go-webauthn`.
        *   `JWT (JSON Web Tokens)`: Sử dụng thư viện `jwx/v2` để xử lý Signing và Encryption.
    *   **CLI:** `Cobra` - Dùng để xây dựng các lệnh quản trị (migrate, serve, user import).

*   **Frontend:**
    *   **Ngôn ngữ:** TypeScript.
    *   **Thư viện UI:** `Preact` (nhẹ hơn React) để xây dựng **Hanko Elements**.
    *   **Công nghệ đóng gói:** `Web Components` - Giúp Hanko Elements có thể chạy trên bất kỳ framework nào (React, Vue, Angular, Svelte) hoặc HTML thuần.

*   **Infrastructure:**
    *   **Docker & Docker Compose**: Để chạy nhanh môi trường Quickstart.
    *   **Kubernetes (K8s) & Skaffold**: Hỗ trợ triển khai Cloud-native quy mô lớn.

---

## 2. Kỹ Thuật và Tư Duy Kiến Trúc (Architecture & Engineering)

Kiến trúc của Hanko được thiết kế theo tư duy **API-First** và **Privacy-First**:

*   **Decoupled Architecture (Kiến trúc tách rời):** Hanko tách biệt hoàn toàn giữa Backend API (`backend/`) và UI Components (`frontend/elements/`). Điều này cho phép nhà phát triển sử dụng bộ giao diện có sẵn hoặc tự xây dựng UI riêng dựa trên `frontend-sdk`.
*   **Mô hình State Machine (Flow API):** Backend không chỉ là các endpoint CRUD đơn thuần mà hoạt động như một cỗ máy trạng thái (State Machine). Các thư mục trong `backend/flow_api` quản lý các trạng thái phức tạp như `credential_onboarding`, `mfa_usage`, giúp xử lý logic xác thực đa bước một cách chặt chẽ.
*   **Data Minimalism:** Chỉ lưu trữ những dữ liệu tối thiểu cần thiết để xác thực, ưu tiên sử dụng các phương thức không mật khẩu (Passkeys) để giảm rủi ro rò rỉ dữ liệu.
*   **Configuration Management:** Sử dụng hệ thống cấu hình phân cấp (YAML, Env Vars) qua thư viện `koanf`. Cho phép ghi đè cấu hình linh hoạt từ file hoặc biến môi trường, phù hợp với mô hình CI/CD hiện đại.

---

## 3. Các Kỹ Thuật Chính Nổi Bật (Key Highlights)

*   **FlowPilot:** Đây là một công nghệ nội bộ (có thể thấy trong `backend/flowpilot`) giúp điều phối các luồng xác thực phức tạp (Flow-based logic). Nó cho phép tùy biến luồng đăng ký/đăng nhập mà không làm hỏng cấu trúc mã nguồn lõi.
*   **Security at Rest (Bảo mật dữ liệu tĩnh):** Hanko không lưu trực tiếp các JSON Web Keys (JWK). Các khóa này được mã hóa bằng **AES-GCM** trước khi lưu vào DB (xem `backend/crypto/aes_gcm`). Khóa master được quản lý qua biến môi trường hoặc AWS KMS.
*   **JWT Customization (Session Templates):** Cho phép người dùng cấu hình các "claims" tùy chỉnh trong JWT thông qua Go templates. Bạn có thể chèn thông tin `metadata` của người dùng vào token một cách động (xem `backend/config/config_session.go`).
*   **Hanko Elements:** Kỹ thuật đóng gói giao diện thành các Web Components giúp nhúng một form đăng nhập Passkey phức tạp vào website chỉ với 2 dòng code: `<hanko-auth>` và `<hanko-profile>`.
*   **Audit Logging chuyên sâu:** Hệ thống ghi lại mọi hành động nhạy cảm (audit logs) và hỗ trợ tự động ẩn danh (masking) thông tin email/username trong log để đảm bảo tuân thủ GDPR.

---

## 4. Tóm Tắt Luồng Hoạt Động (Workflow Summary)

Một quy trình xác thực điển hình trong Hanko diễn ra như sau:

1.  **Khởi tạo (Init):** Giao diện (Hanko Elements) gửi yêu cầu đến Backend để xác định các phương thức xác thực khả dụng cho một định danh (email/username).
2.  **Điều hướng luồng (Flow Navigation):** 
    *   Nếu người dùng có **Passkey**, Backend gửi `Challenge` để trình duyệt ký xác thực (WebAuthn).
    *   Nếu dùng **Passcode**, Backend gọi `mail` service gửi mã OTP 6 số qua SMTP (MailHog/AWS SES).
    *   Nếu dùng **Password**, hệ thống kiểm tra hash mật khẩu (Bcrypt).
3.  **Xử lý trạng thái (State Processing):** `FlowPilot` hoặc `Flow API` sẽ kiểm tra xem người dùng có cần thêm bước MFA (Multi-factor) hay không dựa trên cấu hình trong `config.yaml`.
4.  **Cấp phát phiên (Session Issuance):** Sau khi xác thực thành công, Backend tạo một `Session` trong DB, tạo JWT được ký bởi JWK (từ AWS KMS hoặc Local), và trả về cho Frontend qua Cookie (Set-Cookie) hoặc Header (`X-Auth-Token`).
5.  **Giám sát (Monitoring):** Mọi sự thay đổi (tạo user, đổi email) sẽ kích hoạt **Webhooks** để đồng bộ dữ liệu sang hệ thống bên thứ ba của người dùng.

---

## 5. Kết Luận

Hanko không chỉ là một thư viện đăng nhập mà là một **Identity Infrastructure (Hạ tầng định danh)** hoàn chỉnh. Nó giải quyết bài toán khó nhất của xác thực hiện đại là **Passkeys** và **WebAuthn** một cách đơn giản nhất cho nhà phát triển, đồng thời giữ được tính linh hoạt cực cao thông qua kiến trúc API-centric.