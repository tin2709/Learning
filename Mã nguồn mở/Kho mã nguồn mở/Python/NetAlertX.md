Dựa trên các tệp tin và thông tin mã nguồn bạn cung cấp, dưới đây là bản phân tích chi tiết về **NetAlertX** dưới góc độ kỹ thuật:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)
NetAlertX là một ứng dụng **Hybrid Stack**, kết hợp giữa các ngôn ngữ lập trình truyền thống và hiện đại để tối ưu hóa giữa hiệu suất quét mạng và trải nghiệm người dùng:

*   **Backend Logic (Python 3.12+):** Đây là "trái tim" của hệ thống. Python đảm nhận việc thực hiện các lệnh quét (scans), xử lý dữ liệu thô, chạy các Plugin và quản lý luồng công việc (Workflows). Sử dụng các thư viện như `scapy`, `python-nmap`, và `psutil`.
*   **API Layer (Flask & GraphQL):** Hệ thống sử dụng Flask để cung cấp REST API và Graphene cho GraphQL. Điều này cho phép giao diện người dùng (Frontend) truy vấn dữ liệu một cách linh hoạt và hiệu quả (Read-optimized).
*   **Frontend (PHP 8.3 & JavaScript):** Một điều khá thú vị là giao diện web vẫn sử dụng PHP kết hợp với JavaScript (jQuery, AdminLTE). PHP đóng vai trò là "vỏ bọc" hiển thị, trong khi JavaScript xử lý các tương tác thời gian thực thông qua API và SSE (Server-Sent Events).
*   **Cơ sở dữ liệu (SQLite):** Sử dụng SQLite làm "Nguồn sự thật duy nhất" (Source of Truth). Phù hợp cho môi trường tự lưu trữ (self-hosted) vì không cần cài đặt server DB phức tạp nhưng vẫn đảm bảo tính toàn vẹn dữ liệu.
*   **Networking Tools:** Tận dụng các công cụ hệ thống mạnh mẽ như `arp-scan`, `nmap`, `nbtscan`, `fping` để tương tác với lớp vật lý của mạng.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của NetAlertX được xây dựng theo hướng **Modular & Hardened (Mô-đun hóa & Tăng cường bảo mật)**:

*   **Plugin-Driven Architecture:** Mọi phương thức thu thập dữ liệu (từ UniFi, Pi-hole, SNMP...) đều được tách thành các Plugin riêng biệt. Điều này giúp hệ thống dễ dàng mở rộng mà không làm ảnh hưởng đến mã nguồn cốt lõi.
*   **Hardened Docker Image (Tính bảo mật cao):** 
    *   Hệ thống sử dụng Docker đa giai đoạn (multi-stage builds) dựa trên Alpine Linux để giảm kích thước.
    *   **Nguyên tắc đặc quyền tối thiểu:** Chạy dưới User ID không phải root (20211), sử dụng hệ điều hành Read-only (`read_only: true`), và chỉ cấp các quyền (Capabilities) mạng cần thiết (`NET_ADMIN`, `NET_RAW`).
*   **Source of Truth (NSoT):** NetAlertX không chỉ là công cụ quét, nó đóng vai trò là một kho chứa dữ liệu tài sản mạng. Nó so sánh dữ liệu quét được với dữ liệu đã lưu để phát hiện sự thay đổi (Drift detection).

### 3. Các kỹ thuật chính (Key Techniques)
*   **Heuristics (Phân tích suy luận):** Hệ thống có bộ quy tắc `device_heuristics_rules.json` để tự động đoán loại thiết bị (điện thoại, laptop, IoT) và gán icon dựa trên địa chỉ MAC, Vendor hoặc tên Hostname.
*   **Field Locking (Khóa trường dữ liệu):** Cho phép người dùng "khóa" các thông tin đã chỉnh sửa thủ công để tránh việc các Plugin quét mạng ghi đè lại thông tin sai.
*   **Workflow Engine:** Sử dụng tư duy "Trigger-Condition-Action". Ví dụ: *Nếu* phát hiện thiết bị mới từ Vendor "Apple" *thì* tự động gán vào nhóm "Thiết bị di động".
*   **SSE (Server-Sent Events):** Kỹ thuật đẩy dữ liệu từ Server xuống Client theo thời gian thực giúp giao diện cập nhật trạng thái quét mạng mà không cần tải lại trang (giảm tải 95% so với polling thông thường).
*   **ARP Flux Mitigation:** Kỹ thuật tinh chỉnh tham số hạt nhân (`net.ipv4.conf.all.arp_ignore=1`) để đảm bảo việc quét mạng trong môi trường Docker có nhiều card mạng vật lý đạt độ chính xác cao nhất.

### 4. Tóm tắt luồng hoạt động (Operational Flow)
Hệ thống vận hành theo một vòng lặp kín:

1.  **Discovery (Khám phá):** `Scheduler` (Python) kích hoạt các Plugin quét (ARP-scan, SNMP, UniFi API...).
2.  **Processing (Xử lý):** Dữ liệu thô được đưa về `Backend`. Tại đây, hệ thống thực hiện:
    *   **Matching:** So khớp MAC address để xác định thiết bị cũ hay mới.
    *   **Heuristics:** Nếu là thiết bị mới, tự động đoán loại thiết bị.
    *   **Authoritative Check:** Kiểm tra xem các trường dữ liệu có bị khóa (Locked) hay không trước khi cập nhật.
3.  **Persistence (Lưu trữ):** Cập nhật vào SQLite và ghi log các thay đổi IP, trạng thái Online/Offline.
4.  **Automation (Tự động hóa):** Chạy `Workflows` để phân loại hoặc thực hiện hành động (như Wake-on-LAN).
5.  **Notification (Thông báo):** Nếu có sự thay đổi bất thường (thiết bị lạ, thiết bị quan trọng bị offline), hệ thống gửi thông báo qua Apprise (Telegram, Discord, Email...) hoặc Webhooks.
6.  **Visualization (Hiển thị):** Người dùng theo dõi qua UI (PHP/JS) hoặc truy xuất dữ liệu qua GraphQL/Prometheus Metrics.

### Tổng kết
NetAlertX là một dự án có độ hoàn thiện cao về mặt **bảo mật hệ thống (DevSecOps)** và **tính linh hoạt (Extensibility)**. Dù frontend sử dụng PHP (có vẻ hơi cũ), nhưng phần lõi Python và cách đóng gói Docker lại cực kỳ hiện đại, tập trung vào việc bảo vệ quyền riêng tư và hoạt động hoàn toàn cục bộ (Local-first).