Dựa trên các tệp tin và thông tin bạn cung cấp về dự án **FileKey**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của ứng dụng này:

### 1. Công nghệ cốt lõi (Tech Stack)

FileKey là một ứng dụng web tập trung vào bảo mật tối đa với triết lý "không máy chủ" (Serverless/Client-side only).

*   **Ngôn ngữ lập trình:** Thuần **JavaScript (Vanilla JS)**. Không sử dụng các framework nặng nề để đảm bảo tính minh bạch và dễ kiểm soát mã nguồn.
*   **Web Cryptography API (SubtleCrypto):** Sử dụng các hàm băm và mã hóa cấp thấp có sẵn trong trình duyệt để thực hiện AES-GCM, HKDF và ECDH.
*   **WebAuthn API với Tiện ích mở rộng PRF (Pseudo-Random Function):** Đây là "linh hồn" của ứng dụng. Nó cho phép lấy một giá trị ngẫu nhiên xác định (deterministic) từ Passkey (vân tay, khuôn mặt, hoặc khóa vật lý như Yubikey) để làm gốc cho các khóa mã hóa.
*   **Progressive Web App (PWA):**
    *   `manifest.json`: Định nghĩa cách ứng dụng cài đặt trên màn hình chính.
    *   `sw.js` (Service Worker): Cho phép ứng dụng hoạt động hoàn toàn **offline** sau khi tải lần đầu.
*   **Docker:** Hỗ trợ tự triển khai (self-hosting) thông qua môi trường container hóa (sử dụng Nginx Alpine để phục vụ các tệp tĩnh).

### 2. Các kỹ thuật và tư duy kiến trúc chính

Kiến trúc của FileKey dựa trên ba trụ cột: **Bảo mật tuyệt đối, Quyền riêng tư và Khả năng phục hồi.**

*   **Kiến trúc Zero-Knowledge (Không kiến thức):** Không có tài khoản, không có cơ sở dữ liệu backend. Mọi dữ liệu nhạy cảm và quá trình mã hóa đều diễn ra trong bộ nhớ tạm của trình duyệt người dùng.
*   **Dẫn xuất khóa xác định (Deterministic Key Derivation):** Thay vì lưu trữ khóa mã hóa (nguy hiểm), FileKey tạo lại khóa đó mỗi khi người dùng xác thực bằng Passkey thông qua PRF + HKDF. Điều này có nghĩa là nếu bạn mất máy tính, bạn chỉ cần Passkey để lấy lại quyền truy cập vào các tệp đã mã hóa.
*   **Mã hóa trao đổi khóa Diffie-Hellman (ECDH):** Để chia sẻ tệp mà không cần gửi mật khẩu, ứng dụng sử dụng đường cong Elliptic P-521.
    *   **Share Key:** Thực chất là Khóa công khai (Public Key) được tạo ra từ Passkey của người dùng.
    *   **Bảo mật:** Chỉ người nhận có Khóa riêng tư (Private Key) tương ứng mới có thể giải mã tệp được chia sẻ.
*   **Mã hóa đối xứng AES-256-GCM:** Tiêu chuẩn vàng hiện nay cho mã hóa tệp tin, đảm bảo cả tính bảo mật và tính toàn vẹn (chống sửa đổi tệp).

### 3. Tóm tắt luồng hoạt động (Workflow)

#### A. Luồng Thiết lập & Mã hóa cá nhân:
1.  **Xác thực:** Người dùng đăng ký/đăng nhập bằng Passkey (WebAuthn).
2.  **Tạo mầm (Seed):** Trình duyệt yêu cầu Passkey cung cấp một giá trị PRF.
3.  **Tạo khóa:** Giá trị PRF đi qua thuật toán **HKDF** (với muối ngẫu nhiên 16-byte) để tạo ra khóa mã hóa **AES-256**.
4.  **Mã hóa:** Người dùng thả tệp vào trình duyệt -> Ứng dụng mã hóa tệp bằng khóa vừa tạo -> Người dùng tải xuống tệp đã mã hóa.

#### B. Luồng Chia sẻ tệp (Sharing):
1.  **Người nhận:** Cung cấp "Share Key" (Public ECDH Key) cho người gửi.
2.  **Người gửi:**
    *   Sử dụng Khóa riêng tư (Private ECDH) của mình và Khóa công khai của người nhận.
    *   Tính toán ra một "Shared Secret" (Bí mật chung) qua giao thức ECDH.
    *   Mã hóa tệp bằng bí mật này và gửi tệp `.shared_filekey` cho người nhận.
3.  **Người nhận:** Sử dụng Passkey của mình để tái tạo Khóa riêng tư ECDH -> Kết hợp với Khóa công khai của người gửi (đính kèm trong tệp) -> Tạo ra cùng một "Shared Secret" -> Giải mã tệp.

#### C. Luồng Ngoại tuyến (Offline):
1.  Người dùng truy cập trang web lần đầu.
2.  `Service Worker` tải toàn bộ mã nguồn vào bộ nhớ đệm (Cache).
3.  Lần truy cập sau, dù không có internet, ứng dụng vẫn mở được và thực hiện mã hóa/giải mã bình thường vì mọi tính toán đều nằm ở máy khách (Client-side).

### 💾 Supported Systems

In order to use FileKey, you need a compatible password manager (Apple Passwords, Google Passwords, Windows Hello, etc) or a hardware security key that supports FIDO2 and PRF (like the YubiKey 5 and Bio Series). For hardware security keys, your browser and operating system both need to support WebAuthn and the PRF extension. Below is a non-exhaustive compatiblity table:

| Platform      | Supported Passkey Providers        | Notes               |
|--------------|-------------------------------------|------------------------------------|
| macOS     | Apple Passwords, Yubikey, 1Password         | Safari ≥ 17 or Chrome ≥ 112. Yubikeys will not work in Safari. |
| Windows       | 1Password, YubiKey  | Edge ≥ 112 or Chrome ≥ 112. Requires Windows 11. |
| Linux         | YubiKey (via browser)              | Latest version of Chrome or Chromium-based browsers.  |
| iOS       | Apple Passwords, 1Password | Safari ≥ 17 or Chrome ≥ 112 |
| Android       | Google Passwords, 1Password, Yubikey | Chrome ≥ 112 |

### Kết luận
FileKey không phải là một dịch vụ lưu trữ đám mây, mà là một **công cụ mã hóa đầu cuối (E2EE)** cực kỳ tinh gọn. Nó biến Passkey từ một công cụ chỉ để đăng nhập thành một "chìa khóa vạn năng" để quản lý dữ liệu mà không cần tin tưởng vào bất kỳ bên thứ ba nào.