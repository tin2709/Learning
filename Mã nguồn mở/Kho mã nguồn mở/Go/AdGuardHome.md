Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **AdGuard Home** mà bạn cung cấp, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống.

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

#### Backend (Ngôn ngữ Go - Golang)
*   **DNS Engine:** Sử dụng các thư viện chuyên biệt như `miekg/dns` (xử lý giao thức DNS cơ bản), `AdguardTeam/dnsproxy` (để tạo proxy DNS) và `AdguardTeam/urlfilter` (để lọc URL hiệu năng cao).
*   **Giao thức hỗ trợ:** Hỗ trợ đầy đủ các tiêu chuẩn hiện đại: DNS-over-HTTPS (DoH), DNS-over-TLS (DoT), DNS-over-QUIC (DoQ), và DNSCrypt.
*   **Hệ thống lưu trữ:** Sử dụng `bbolt` (Key/Value store thuần Go) để lưu trữ cấu hình và dữ liệu phiên làm việc, đảm bảo tốc độ và tính di động.
*   **Networking:** Tương tác trực tiếp với lớp mạng để quản lý DHCP (IPv4/IPv6), IPSet (trên Linux để chặn IP ở tầng kernel), và ARP/neighbor tables.

#### Frontend (React.js & TypeScript)
*   **Framework:** React 16+ kết hợp với Redux để quản lý trạng thái (state management).
*   **UI/UX:** Sử dụng Tabler (dựa trên Bootstrap/CSS) để tạo giao diện quản trị hiện đại.
*   **Build Tool:** Webpack được cấu hình phức tạp để tối ưu hóa bundle và hỗ trợ môi trường phát triển (hot reload).
*   **Đa ngôn ngữ:** Hệ thống `i18next` kết hợp với dịch vụ `CrowdIn` để hỗ trợ hàng chục ngôn ngữ.

#### Quản lý dự án & Build
*   **CI/CD:** Cấu hình Bamboo (`bamboo.yaml`) và GitHub Actions để tự động hóa việc build, test và phát hành.
*   **Packaging:** Hỗ trợ Docker, Snapcraft và các bản build standalone cho hầu hết các kiến trúc CPU (amd64, arm64, mips, v.v.).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

#### Kiến trúc "All-in-One" và Tự trị
AdGuard Home được thiết kế để chạy như một binary duy nhất (Single Binary) tích hợp cả:
*   DNS Server (Engine lõi).
*   Web Server (Giao diện quản trị).
*   DHCP Server (Cấp phát IP).
*   Hệ thống cập nhật tự động.

#### Cơ chế Nhúng (Embedding)
Một điểm đặc biệt trong kiến trúc Go là việc sử dụng `go:embed` (trong `main.go`). Toàn bộ mã nguồn React sau khi build sẽ được nén và nhúng trực tiếp vào tệp thực thi Go. Điều này giúp người dùng cuối chỉ cần tải 1 file duy nhất về chạy mà không cần cài đặt môi trường Node.js hay Web server riêng lẻ.

#### Tư duy Filter-Chain (Chuỗi lọc)
Luồng xử lý yêu cầu được thiết kế theo dạng pipeline (đường ống):
1.  **Tiền xử lý:** Kiểm tra danh sách truy cập (Access Control), kiểm tra Rewrite.
2.  **Bộ lọc tĩnh:** Đối khớp tên miền với hàng triệu quy tắc trong bộ lọc (Adblock rules).
3.  **Dịch vụ an toàn:** Gọi các API bảo vệ (Safe Browsing, Parental Control) bằng cơ chế hash prefix để bảo vệ quyền riêng tư của người dùng (không gửi trực tiếp domain lên server AdGuard).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **DNS Sinkholing:** Kỹ thuật trả về IP "hố đen" (0.0.0.0) hoặc NXDOMAIN cho các yêu cầu đến tên miền quảng cáo/theo dõi.
*   **Optimistic Caching:** Kỹ thuật trả về kết quả DNS đã hết hạn từ cache trước, sau đó mới cập nhật cache ngầm để giảm độ trễ tối đa cho người dùng.
*   **Fastest IP:** Gửi yêu cầu DNS đến nhiều upstream cùng lúc, đo tốc độ phản hồi TCP và chọn kết quả từ server nhanh nhất.
*   **Anonymization:** Kỹ thuật xóa bớt các bit cuối của địa chỉ IP khách hàng (Client IP) trước khi lưu vào log để đảm bảo tuân thủ GDPR/quyền riêng tư.
*   **Rate Limiting:** Giới hạn tần suất yêu cầu trên mỗi client để chống tấn công DNS Amplification.

---

### 4. Tóm tắt luồng hoạt động (README.vn)

Dưới đây là nội dung tóm tắt dự án bằng tiếng Việt theo phong cách README:

# AdGuard Home - Trung tâm bảo vệ mạng gia đình

AdGuard Home là một phần mềm quản lý DNS toàn mạng, giúp chặn quảng cáo và mã độc mà không cần cài đặt ứng dụng trên từng thiết bị.

## 🚀 Luồng hoạt động chính
1.  **Tiếp nhận yêu cầu:** Thiết bị trong mạng (điện thoại, TV, laptop) gửi yêu cầu DNS đến AdGuard Home.
2.  **Kiểm tra bộ lọc:** 
    *   Nếu domain nằm trong danh sách chặn -> Trả về IP trống (0.0.0.0).
    *   Nếu có quy tắc Rewrite (Chuyển hướng) -> Trả về IP đích đã cấu hình.
3.  **Bảo vệ thông minh:** Kiểm tra xem domain có chứa nội dung người lớn (Parental Control) hay lừa đảo (Safe Browsing) không.
4.  **Truy vấn Upstream:** Nếu domain "sạch", AdGuard Home sẽ hỏi các DNS server cấp trên (như Google, Cloudflare) qua các giao thức bảo mật (DoH, DoT).
5.  **Trả kết quả & Ghi log:** Lưu kết quả vào bộ nhớ đệm (Cache) và hiển thị lịch sử truy vấn lên giao diện web.

## 🛠 Các thành phần cốt lõi
*   **DNS Server:** Trái tim của hệ thống, xử lý mọi yêu cầu mạng.
*   **Web Admin:** Giao diện React cực kỳ trực quan để quản lý.
*   **DHCP Server:** Tự động cấp IP và ép mọi thiết bị mới phải dùng DNS của AdGuard.
*   **Bộ lọc tùy chỉnh:** Hỗ trợ cả định dạng `/etc/hosts` và cú pháp chặn nâng cao của Adblock.

## 🛡 Tại sao nên dùng?
*   **Kiểm soát hoàn toàn:** Bạn là người quyết định cái gì được phép vào mạng nhà mình.
*   **Hiệu năng cao:** Viết bằng Go, tiêu tốn cực ít RAM và CPU (phù hợp chạy trên Raspberry Pi, Router).
*   **Riêng tư tuyệt đối:** Dữ liệu không gửi đi đâu cả, mọi thứ nằm trong thiết bị của bạn.

---
*Tài liệu này được phân tích dựa trên mã nguồn phiên bản AdGuard Home 2025.*