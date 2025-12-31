Dưới đây là bản phân tích chi tiết và file README tóm tắt dự án **Pocket ID** bằng tiếng Việt, tập trung vào khía cạnh kỹ thuật và kiến trúc.

---

# Phân Tích Kỹ Thuật Dự Án Pocket ID

## 1. Công Nghệ Cốt Lõi (Core Technologies)
*   **Backend (Go):** Sử dụng ngôn ngữ Go mang lại hiệu suất cao và khả năng đóng gói thành file thực thi duy nhất.
    *   **Gin Framework:** Web framework để xử lý các yêu cầu HTTP/API.
    *   **GORM:** Thư viện ORM để tương tác với cơ sở dữ liệu (hỗ trợ SQLite và PostgreSQL).
    *   **WebAuthn (Passkeys):** Thư viện Go-WebAuthn để xử lý đăng nhập không mật khẩu.
    *   **OIDC:** Triển khai giao thức OpenID Connect tiêu chuẩn.
    *   **Cobra:** Xử lý các câu lệnh CLI (import/export, key-rotate).
    *   **OpenTelemetry:** Tích hợp quan sát (tracing, metrics, logging).
*   **Frontend (SvelteKit):** Sử dụng SvelteKit giúp ứng dụng nhanh, nhẹ và tối ưu hóa phía client.
    *   **TypeScript:** Đảm bảo kiểu dữ liệu an toàn.
    *   **Tailwind CSS & Shadcn/UI (Svelte version):** Xây dựng giao diện hiện đại, dễ tùy biến.
*   **DevOps & Tooling:** Docker, pnpm (monorepo), GitHub Actions.

## 2. Tư Duy Kiến Trúc (Architectural Thinking)
*   **Layered Architecture (Kiến trúc phân lớp):**
    *   `Controller`: Tiếp nhận request, validate dữ liệu thông qua DTO.
    *   `Service`: Chứa logic nghiệp vụ chính (Business Logic).
    *   `Model`: Định nghĩa cấu trúc dữ liệu trong DB.
    *   `DTO (Data Transfer Object)`: Lớp trung gian để giao tiếp với Frontend, giúp bảo mật dữ liệu internal.
*   **Storage Abstraction:** Pocket ID trừu tượng hóa việc lưu trữ file (hình ảnh, cấu hình) thông qua interface `FileStorage`, cho phép chuyển đổi linh hoạt giữa File System, S3 (AWS/Minio) hoặc lưu trực tiếp trong Database.
*   **Security-First:** Loại bỏ hoàn toàn mật khẩu (Passwordless). Chỉ sử dụng Passkeys giúp ngăn chặn các cuộc tấn công lừa đảo (Phishing) và rò rỉ mật khẩu.
*   **Single Binary:** Frontend được biên dịch và nhúng trực tiếp vào file thực thi Go (sử dụng `embed` package), giúp việc triển khai cực kỳ đơn giản (chỉ 1 file duy nhất).

## 3. Các Kỹ Thuật Chính (Key Techniques)
*   **Unicode Normalization:** Sử dụng `unorm` tags trong DTO để chuẩn hóa dữ liệu đầu vào (NFC/NFD), tránh lỗi định dạng chuỗi giữa các nền tảng.
*   **Rate Limiting & Security Middleware:** Tích hợp giới hạn tần suất yêu cầu (rate limit) cho các endpoint nhạy cảm và áp dụng các chính sách bảo mật như CSP (Content Security Policy) với Nonce.
*   **Database Migrations:** Tự động cập nhật cấu trúc database thông qua script migration cho cả Postgres và SQLite.
*   **Observability (Slog Fanout):** Kỹ thuật ghi log đồng thời ra console (dạng human-readable) và OpenTelemetry (dạng JSON cho máy quét) bằng cách tùy biến `slog.Handler`.

---

# Tóm tắt Dự Án (README_VN.md)

# Pocket ID - Nhà cung cấp định danh OIDC qua Passkeys

Pocket ID là một giải pháp định danh OIDC (OpenID Connect) đơn giản, cho phép người dùng xác thực vào các dịch vụ tự lưu trữ (self-hosted) bằng **Passkeys** thay vì mật khẩu truyền thống.

## 🚀 Luồng hoạt động của Hệ thống

### 1. Đăng ký & Thiết lập (Setup Flow)
-   **Khởi tạo:** Khi chạy lần đầu, hệ thống yêu cầu tạo người dùng Admin đầu tiên.
-   **Đăng ký Passkey:** Người dùng sử dụng thiết bị (Yubikey, FaceID, TouchID, Windows Hello) để tạo một cặp khóa mật mã. Khóa công khai (Public Key) được lưu vào DB của Pocket ID.

### 2. Xác thực người dùng (Authentication Flow)
-   **Yêu cầu đăng nhập:** Khi người dùng truy cập một ứng dụng (ví dụ: Nextcloud), ứng dụng đó chuyển hướng đến Pocket ID.
-   **Thử thách WebAuthn:** Pocket ID gửi một "thử thách" (challenge) về trình duyệt.
-   **Ký xác thực:** Người dùng xác nhận trên thiết bị, thiết bị ký vào thử thách bằng khóa bí mật (Private Key) và gửi lại cho Pocket ID.
-   **Kiểm tra:** Pocket ID dùng khóa công khai để xác minh chữ ký. Nếu khớp, người dùng được đăng nhập thành công.

### 3. Cấp quyền OIDC (OIDC Authorization Flow)
-   Sau khi xác thực, Pocket ID cấp một **Authorization Code**.
-   Ứng dụng đích trao đổi mã này lấy **Access Token**, **ID Token** và **Refresh Token**.
-   Ứng dụng sử dụng token để lấy thông tin người dùng (Email, Tên, Nhóm) từ endpoint `/userinfo`.

## 🛠 Kiến trúc thư mục chính

*   `/backend`: Chạy bằng Go (Gin, GORM).
    *   `/internal/service`: Trái tim của hệ thống, xử lý logic OIDC, WebAuthn, LDAP.
    *   `/internal/controller`: Các API endpoint.
    *   `/resources/migrations`: Script quản lý phiên bản database.
*   `/frontend`: Chạy bằng SvelteKit + TypeScript.
    *   `/src/lib/components`: Thư viện UI xây dựng theo phong cách Shadcn.
*   `/docker`: Chứa Dockerfile để đóng gói ứng dụng.

## 🌟 Tính năng nổi bật

-   **Hoàn toàn không mật khẩu:** Bảo mật tuyệt đối với WebAuthn.
-   **Hỗ trợ LDAP:** Đồng bộ người dùng và nhóm từ các máy chủ LDAP hiện có.
-   **Giao diện tùy biến:** Thay đổi logo, màu sắc, hình nền ngay trên bảng điều khiển.
-   **Ghi nhật ký kiểm tra (Audit Logs):** Theo dõi mọi hoạt động đăng nhập, thay đổi cấu hình kèm thông tin vị trí (GeoIP).
-   **Đa ngôn ngữ:** Hỗ trợ nhiều ngôn ngữ bao gồm tiếng Việt.

## 📦 Cài đặt nhanh

Sử dụng Docker Compose:

```yaml
services:
  pocket-id:
    image: ghcr.io/pocket-id/pocket-id:latest
    ports:
      - 1411:1411
    volumes:
      - ./data:/app/data
    environment:
      - APP_URL=https://auth.yourdomain.com
```

---
**Pocket ID** - *Passkeys là tương lai, và chúng tôi mang tương lai đó đến với các dịch vụ self-hosted của bạn.*