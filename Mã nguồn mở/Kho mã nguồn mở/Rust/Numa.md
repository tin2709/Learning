Dưới đây là bản phân tích chi tiết về dự án **Numa** — một DNS resolver được viết từ đầu (from scratch) bằng Rust, dựa trên nội dung mã nguồn và các bài viết kỹ thuật đi kèm.

---

### 1. Công nghệ cốt lõi (Core Stack)

Dự án tận dụng hệ sinh thái hiện đại của Rust để đạt hiệu suất và độ an toàn cao:

*   **Ngôn ngữ:** Rust (phiên bản 1.94+ trong Dockerfile). Không sử dụng thư viện DNS bên thứ ba (`hickory-dns` hay `trust-dns`), tự triển khai giao thức ở mức byte.
*   **Asynchronous Runtime:** `tokio` — Xử lý hàng nghìn truy vấn đồng thời với mô hình non-blocking I/O.
*   **Web Framework & API:** `axum` & `hyper` — Dùng cho Dashboard quản trị và REST API.
*   **Cryptography:** `ring` và `rustls` — Thực hiện các thuật toán mã hóa cho DNSSEC (RSA, ECDSA, Ed25519) và bảo mật kênh truyền (DoT, DoH).
*   **Giao thức hỗ trợ:**
    *   **UDP/TCP DNS:** Giao thức truyền thống (RFC 1035).
    *   **DoH (DNS-over-HTTPS):** RFC 8484.
    *   **DoT (DNS-over-TLS):** RFC 7858.
    *   **ODoH (Oblivious DoH):** RFC 9230 (Tách biệt IP người dùng và nội dung truy vấn qua Relay).
    *   **mDNS:** Khám phá dịch vụ trong mạng LAN.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Numa được xây dựng theo hướng **"All-in-one"** nhưng vẫn đảm bảo tính module hóa:

*   **Mô hình Pipeline (Resolution Pipeline):** Mọi truy vấn đi qua một chuỗi các bước kiểm tra logic:
    `Query` → `Overrides` → `.numa TLD` → `Ad-block` → `Zones` → `Cache` → `Upstream (Forward/Recursive)`.
*   **Thiết kế Zero-dependency DNS:** Việc tự viết bộ parser/serializer giúp tác giả kiểm soát hoàn toàn bộ đệm (buffer), tối ưu hóa việc nén nhãn (label compression) và quản lý con trỏ (pointer) trong gói tin DNS.
*   **Local-First & Portable:** Numa không chỉ là một server mà còn là một công cụ hỗ trợ phát triển (local dev). Nó tích hợp sẵn Proxy ngược (Reverse Proxy) và tự động cấp chứng chỉ TLS cho các tên miền `.numa`.
*   **Kháng lỗi (Resilience):** Áp dụng triết lý "The Tail at Scale" của Google để giảm độ trễ đuôi (tail latency) thông qua kỹ thuật **Request Hedging** (gửi nhiều yêu cầu song song đến các thượng nguồn khác nhau và lấy kết quả nhanh nhất).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Quản lý Cache thông minh:**
    *   **Lazy Eviction:** Thay vì dùng thread chạy ngầm để xóa cache hết hạn, Numa tính toán TTL còn lại khi đọc (`remaining = original_ttl - elapsed`).
    *   **Serve-stale (RFC 8767):** Trả về kết quả cũ đã hết hạn (với TTL=1) trong khi cập nhật lại cache ở background để đảm bảo người dùng không phải chờ.
*   **Xử lý DNSSEC từ đầu:**
    *   Xây dựng chuỗi tin cậy (Chain of Trust) từ Root Anchor.
    *   Triển khai **NSEC/NSEC3** để chứng minh sự không tồn tại của một tên miền một cách an toàn về mặt mật mã.
*   **Tối ưu hóa độ trễ:**
    *   **SRTT (Smoothed Round Trip Time):** Theo dõi hiệu suất của các nameserver để ưu tiên chọn server nhanh nhất.
    *   **TLD Priming:** Tự động tải thông tin của các TLD phổ biến (`.com`, `.net`,...) khi khởi động để bỏ qua bước truy vấn Root Server trong lần đầu.
*   **Auto-TLS & On-the-fly CA:** Tự động tạo Certificate Authority (CA) nội bộ và cấp chứng chỉ cho các dịch vụ local, tích hợp sẵn quy trình cài đặt lên iPhone/Android qua mã QR.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Tiếp nhận (Ingest):** Numa lắng nghe trên cổng 53 (UDP/TCP), 853 (DoT) hoặc thông qua endpoint HTTPS (DoH).
2.  **Lọc & Ghi đè (Intercept):**
    *   Kiểm tra danh sách chặn (385K+ domains). Nếu khớp, trả về `0.0.0.0` (hoặc NXDOMAIN).
    *   Kiểm tra Overrides: Nếu là `frontend.numa`, chuyển hướng truy vấn đến Proxy nội bộ đang map với một cổng cụ thể (ví dụ `:5173`).
3.  **Tra cứu (Lookup):**
    *   Tìm trong cache. Nếu có (Fresh) hoặc cũ nhưng chấp nhận được (Stale), trả về ngay.
    *   Nếu không có, thực hiện truy vấn thượng nguồn:
        *   *Mode Forward:* Gửi đến Quad9/Cloudflare qua DoH/DoT/UDP.
        *   *Mode Recursive:* Tự đi hỏi từ Root Server xuống đến Authoritative Server.
4.  **Xác thực (Validate):** Nếu chạy ở chế độ Recursive, Numa thực hiện kiểm tra chữ ký DNSSEC để đảm bảo dữ liệu không bị giả mạo.
5.  **Phản hồi & Lưu trữ (Respond & Record):** Trả kết quả cho client, cập nhật cache, điều chỉnh TTL và ghi log truy vấn để hiển thị trên Dashboard.

---

**Kết luận:** Numa là một minh chứng cho sức mạnh của Rust trong việc xây dựng hạ tầng mạng thấp cấp. Nó biến DNS từ một dịch vụ hạ tầng khô khan thành một công cụ năng suất cho lập trình viên, kết hợp giữa quyền riêng tư (ODoH), tốc độ (Hedging) và tiện ích (Local Services).