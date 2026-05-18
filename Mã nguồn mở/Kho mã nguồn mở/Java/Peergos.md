Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp về **Peergos**, đây là một dự án cực kỳ tham vọng và phức tạp về mặt kỹ thuật, nhằm xây dựng một "Web riêng tư" (Private Web).

Dưới đây là phân tích chi tiết:

### 1. Công nghệ Cốt lõi (Core Technologies)
*   **Ngôn ngữ chính:** **Java (99.9%)**. Dự án sử dụng Java cho cả phía Server và phía Client.
*   **GWT (Google Web Toolkit):** Đây là điểm độc đáo nhất. Toàn bộ logic nghiệp vụ phức tạp phía Client được viết bằng Java và biên dịch chéo (cross-compile) sang JavaScript để chạy trên trình duyệt.
*   **IPFS & Nabu:** Peergos sử dụng IPFS làm lớp lưu trữ và truyền tải dữ liệu ngang hàng (P2P). Họ tự phát triển **Nabu**, một bản thực thi IPFS tối giản bằng Java để tối ưu hiệu suất.
*   **Mật mã học (Cryptography):**
    *   **TweetNaCl:** Sử dụng bản Java/JS cho các thuật toán Salsa20, Poly1305, Curve25519.
    *   **Post-Quantum (PQ):** Tích hợp thuật toán **ML-KEM** (Kyber) để chống lại các cuộc tấn công từ máy tính lượng tử trong tương lai.
    *   **scrypt:** Dùng để tạo khóa từ mật khẩu người dùng với độ bảo mật cực cao.
*   **Lưu trữ Metadata:** Hỗ trợ cả SQLite (cho cá nhân) và PostgreSQL (cho hệ thống lớn).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Peergos được thiết kế theo mô hình **Zero-Trust (Không tin cậy bất kỳ ai)**:
*   **Trustless Servers:** Các máy chủ (Peergos instances) hoàn toàn không có quyền truy cập vào dữ liệu người dùng. Mọi thứ được mã hóa đầu cuối (E2EE). Máy chủ chỉ thấy các khối dữ liệu (chunks) bị xáo trộn và không biết tên tệp, kích thước tệp hay cấu trúc thư mục.
*   **Cấu trúc 6 lớp:**
    1.  *Data Layer (IPFS):* Lưu trữ thô.
    2.  *Auth Layer:* Cặp khóa điều khiển quyền ghi (mọi thay đổi phải có chữ ký).
    3.  *Merkle-CHAMP:* Cấu trúc dữ liệu để quản lý các mảng dữ liệu lớn một cách bảo mật.
    4.  *Encryption Layer:* Chia tệp thành các khối 5MiB và mã hóa độc lập.
    5.  *Social Layer:* Quản lý quan hệ bạn bè mà không để lộ biểu đồ xã hội (Friendship graph).
    6.  *Sharing Layer:* Hệ thống **Cryptree** để chia sẻ quyền truy cập tệp theo phân cấp.
*   **Tính độc lập:** Không phụ thuộc vào hệ thống CA (Certificate Authority) truyền thống của TLS, thay vào đó sử dụng cấu trúc tương tự Certificate Transparency để xác thực tên người dùng duy nhất.

### 3. Kỹ thuật Lập trình Đặc sắc (Notable Techniques)
*   **Giả lập thư viện chuẩn (JRE Emulation):** Trong thư mục `src/peergos/gwt/emu`, Peergos đã viết lại (giả lập) một phần lớn thư viện chuẩn của Java (`java.io`, `java.nio`, `java.time`, `java.util.concurrent`) để chúng có thể chạy được trên trình duyệt thông qua GWT. Điều này cho phép họ dùng chung 90% mã nguồn giữa Server và Client.
*   **Cryptree mở rộng:** Tác giả mở rộng hệ thống Cryptree để bảo vệ không chỉ nội dung mà cả siêu dữ liệu (metadata) như kích thước tệp, thumbnail, và cấu trúc thư mục.
*   **Cơ chế Đăng nhập không truyền khóa:** Sử dụng `scrypt` với muối (salt) và hash mật khẩu để tạo ra khóa đối xứng và cặp khóa ký ngay tại máy khách. Người dùng có thể đăng nhập từ bất kỳ máy nào mà không bao giờ phải gửi khóa bí mật đi qua mạng.
*   **Blinded Follow Requests:** Kỹ thuật làm mù (blinding) trong các yêu cầu kết bạn, khiến máy chủ không thể biết ai đang gửi yêu cầu cho ai, bảo vệ sự riêng tư của mạng lưới quan hệ.

### 4. Luồng Hoạt động Hệ thống (System Workflow)
1.  **Luồng Đăng nhập (Login):**
    *   Username + Password -> `scrypt` -> Khóa đối xứng (Symmetric) + Cặp khóa ký (Signing).
    *   Dùng khóa ký để xác thực với server và tải về "Login Data" đã mã hóa.
    *   Dùng khóa đối xứng giải mã "Login Data" để lấy khóa định danh (Identity keys) và quyền truy cập thư mục gốc (Root capability).
2.  **Luồng Lưu trữ (Storage):**
    *   Tệp tin được chia thành các khối 5MiB -> Mã hóa từng khối -> Gắn nhãn ngẫu nhiên -> Đẩy lên IPFS/S3.
    *   Mỗi tệp/thư mục có 2 khóa đối xứng ngẫu nhiên (không dùng mã hóa hội tụ để tránh rò rỉ thông tin).
3.  **Luồng Chia sẻ (Sharing):**
    *   Người dùng tạo một liên kết bí mật (Secret link) có dạng `https://domain/#KEY_MATERIAL`.
    *   Phần sau dấu `#` không bao giờ được gửi lên máy chủ. Trình duyệt phía người nhận sẽ dùng `KEY_MATERIAL` để tự giải mã dữ liệu ngay tại máy cục bộ.
4.  **Luồng Di trú (Migration):**
    *   Người dùng có thể chạy lệnh `migrate` để di chuyển toàn bộ dữ liệu từ server này sang server khác. Do định danh là mật mã học (cryptographic identity), mọi liên kết và quan hệ xã hội vẫn giữ nguyên giá trị.

**Tổng kết:** Peergos là một kiệt tác về kỹ thuật Java, kết hợp nhuần nhuyễn giữa hệ thống phân tán (P2P) và mật mã học tiên tiến. Việc họ tự xây dựng lớp giả lập JRE để chạy trên Web cho thấy trình độ kiểm soát mã nguồn ở mức cực kỳ sâu.