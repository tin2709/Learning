Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Kanidm**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật lập trình và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Kanidm được xây dựng như một giải pháp Identity Management (IDM) hiện đại, thay thế cho các hệ thống cũ như FreeIPA hay OpenLDAP.

*   **Ngôn ngữ lập trình:**
    *   **Rust:** Chiếm >70% mã nguồn. Tận dụng tính an toàn bộ nhớ (memory safety) và hiệu năng cao để xử lý các tác vụ nhạy cảm về bảo mật.
    *   **Python:** Sử dụng chủ yếu cho thư viện client (`pykanidm`) và các module mở rộng như RADIUS (`rlm_python`).
*   **Lưu trữ dữ liệu:**
    *   **SQLite:** Không dùng như một cơ sở dữ liệu quan hệ truyền thống mà dùng như một **Key-Value store** bền vững. Kanidm tự xây dựng lớp Indexing, Caching và Query Optimization riêng bên trên.
*   **Giao thức bảo mật & Định danh:**
    *   **WebAuthn (Passkeys):** Hỗ trợ mạnh mẽ xác thực không mật khẩu và Attestation (xác thực thiết bị phần cứng).
    *   **OAuth2 / OpenID Connect (OIDC):** Giao thức hiện đại cho ứng dụng web/SaaS.
    *   **LDAP (Read-only):** Cung cấp cổng kết nối cho các hệ thống di sản (legacy systems).
    *   **RADIUS:** Dùng cho xác thực mạng (Wi-Fi, VPN).
*   **Mạng & Hạ tầng:**
    *   **Tokio & Axum:** Runtime async và web framework hiện đại của Rust để xử lý hàng ngàn kết nối đồng thời.
    *   **OpenTelemetry:** Tích hợp sẵn để giám sát (tracing) và đo lường hiệu năng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Kanidm tuân thủ các nguyên tắc bảo mật nghiêm ngặt:

*   **Security-by-Design (Bảo mật từ thiết kế):**
    *   **Bắt buộc TLS:** Hệ thống từ chối hoạt động nếu không có mã hóa TLS. Ngay cả giữa Load Balancer và Server cũng yêu cầu HTTPS để đảm bảo "Zero Trust".
    *   **Secure-by-Default:** Các thiết lập mặc định luôn là an toàn nhất (ví dụ: Secure cookies, HSTS).
*   **Mô hình Consistency (PA trong CAP Theorem):**
    *   Kanidm ưu tiên tính **Sẵn sàng (Availability)** và **Khả năng chịu lỗi phân vùng (Partition Tolerance)**.
    *   Sử dụng cơ chế **Replication (Eventual Consistency)** thay vì các giao thức đồng thuận chặt chẽ như Raft để giảm độ trễ khi triển khai quy mô toàn cầu.
*   **Phân tách đặc quyền (Privilege Separation):**
    *   Phân chia rõ ràng giữa `admin` (quản trị hệ thống) và `idm_admin` (quản trị người dùng).
    *   Cơ chế **Reauthentication (Privilege Access Mode)**: Người dùng phải xác thực lại khi thực hiện các tác vụ ghi nhạy cảm (tương tự `sudo`).
*   **Tính module hóa cao:**
    *   Dự án chia thành nhiều crate (thư viện): `kanidmd_lib` (lõi DB), `kanidm_proto` (giao thức), `kanidm_client` (thư viện kết nối).

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Strong Typing & Newtypes:** Sử dụng hệ thống kiểu dữ liệu của Rust để ngăn chặn các lỗi logic (ví dụ: Phân biệt rõ giữa UUID của User và UUID của Group ở mức compiler).
*   **Procedural Macros:** Sử dụng macro để giảm mã lặp (boilerplate), đặc biệt trong việc định nghĩa Schema và các thao tác CRUD trên database (`kanidmd_lib_macros`).
*   **Memory Management:** Sử dụng các bộ cấp phát bộ nhớ hiệu năng cao như `mimalloc` hoặc `jemalloc` để tối ưu hóa throughput.
*   **Async/Await:** Tận dụng tối đa mô hình xử lý bất đồng bộ để không chặn (blocking) các luồng xử lý khi thực hiện I/O hoặc truy vấn DB.
*   **HJSON cho Di trú dữ liệu (Migration):** Sử dụng định dạng HJSON (JSON cho phép comment) để quản lý cấu hình người dùng/nhóm dưới dạng mã (Identity as Code).

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Luồng xác thực (Authentication Flow)
1.  **Khởi tạo (Intent):** Client yêu cầu đăng nhập. Server tạo một "Intent Session" và gửi về một Secure Cookie.
2.  **Thách thức (Challenge):** Server kiểm tra chính sách tài khoản (Account Policy) và gửi về các phương thức xác thực cho phép (Password, TOTP, WebAuthn).
3.  **Xác thực:** Người dùng cung cấp bằng chứng (credentials). Server xác minh và nâng cấp session lên trạng thái xác thực thành công.
4.  **Cấp Token:** Server cấp UAT (User Authentication Token) cho client để sử dụng trong các yêu cầu tiếp theo.

#### B. Luồng tích hợp Unix (Unix Integration Workflow)
1.  **Yêu cầu:** Người dùng thực hiện `ls -l` hoặc đăng nhập SSH trên máy Linux.
2.  **NSS/PAM:** Thư viện `nss_kanidm` hoặc `pam_kanidm` được gọi.
3.  **Unixd:** Yêu cầu được gửi qua Unix Socket đến daemon cục bộ `kanidm_unixd`.
4.  **Server:** `kanidm_unixd` truy vấn Server Kanidm (hoặc bộ nhớ cache cục bộ) để trả về thông tin UID/GID và quyền truy cập.

#### C. Luồng đồng bộ hóa (Replication & Sync)
1.  **Thay đổi:** Một thay đổi xảy ra trên Node A.
2.  **Changelog:** Node A ghi lại thay đổi vào nhật ký hệ thống.
3.  **Pull/Push:** Các Node khác trong Topology định kỳ kiểm tra và kéo (pull) các thay đổi này về để áp dụng vào DB cục bộ của mình, đảm bảo tính hội tụ dữ liệu.

### Tóm lược
Dự án **Kanidm** là một minh chứng điển hình cho việc áp dụng **Rust** vào lĩnh vực hạ tầng bảo mật. Nó ưu tiên sự an toàn tuyệt đối và hiệu năng thông qua việc tự xây dựng các lớp xử lý dữ liệu thay vì phụ thuộc hoàn toàn vào các bên thứ ba, đồng thời chuẩn hóa mọi giao tiếp qua giao thức mã hóa hiện đại.