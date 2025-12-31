Dưới đây là bản phân tích chi tiết về dự án **BillionMail** dựa trên cấu trúc mã nguồn và các tệp tin cấu hình mà bạn đã cung cấp.

---

# BillionMail: Phân tích Hệ thống Mail Server & Marketing Mã nguồn mở

BillionMail là một giải pháp tự lưu trữ (self-hosted) toàn diện, kết hợp giữa một Mail Server truyền thống và một nền tảng Email Marketing hiện đại.

## 1. Phân tích Công nghệ Cốt lõi (Tech Stack)

Dự án được xây dựng trên mô hình **Containerization** (Container hóa) với các thành phần chuyên biệt:

*   **Ngôn ngữ lập trình chính:**
    *   **Go (Golang):** Chiếm >92%, sử dụng framework **GoFrame**. Đây là "trái tim" của hệ thống (`core-billionmail`), điều phối toàn bộ logic nghiệp vụ, API và quản lý chiến dịch.
    *   **Shell Script:** Dùng cho quá trình cài đặt (`install.sh`), cập nhật (`update.sh`) và quản lý dòng lệnh (`bm.sh`).
    *   **Vue.js/TypeScript (Frontend):** Được tìm thấy trong thư mục `core/frontend`, dùng để xây dựng bảng điều khiển (Dashboard) quản trị.
*   **Hệ hạ tầng Email:**
    *   **Postfix:** Đóng vai trò MTA (Mail Transfer Agent) - xử lý việc gửi và nhận thư qua giao thức SMTP.
    *   **Dovecot:** Đóng vai trò MDA (Mail Delivery Agent) - quản lý lưu trữ hòm thư, hỗ trợ giao thức IMAP/POP3.
    *   **Rspamd:** Hệ thống lọc thư rác (Spam filter) tiên tiến, tích hợp học máy và kiểm tra chữ ký số.
    *   **RoundCube:** Trình duyệt webmail mã nguồn mở cho người dùng cuối.
*   **Lưu trữ & Dữ liệu:**
    *   **PostgreSQL:** Cơ sở dữ liệu quan hệ chính lưu trữ thông tin cấu hình, hòm thư, danh sách khách hàng và nhật ký chiến dịch.
    *   **Redis:** Dùng làm bộ nhớ đệm (Caching), hàng đợi (Queue) cho các tác vụ gửi thư hàng loạt và lưu trữ dữ liệu thống kê cho Rspamd.

## 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của BillionMail được thiết kế theo hướng **Modular Micro-services** thông qua Docker:

1.  **Sự tách biệt hoàn toàn (Separation of Concerns):** Mỗi dịch vụ (SMTP, IMAP, Database, Logic core) chạy trong một container riêng biệt. Điều này giúp hệ thống ổn định; nếu Webmail lỗi, việc gửi thư tự động (Campaign) vẫn hoạt động bình thường.
2.  **Thiết kế hướng API:** Phần `core` cung cấp RESTful API toàn diện (v1) để quản lý mọi thứ từ tên miền, hòm thư đến các chiến dịch gửi email tỷ lệ lớn.
3.  **Khả năng mở rộng (Scalability):** Sử dụng Redis làm hàng đợi cho phép BillionMail xử lý hàng triệu email mà không làm nghẽn hệ thống xử lý chính.
4.  **Tích hợp AI (Future-ready):** Dự án có sẵn các template cấu hình cho OpenAI, Anthropic, DeepSeek, Gemini... cho thấy tư duy ứng dụng AI vào việc soạn thảo nội dung email marketing hoặc tối ưu hóa chiến dịch.

## 3. Các Kỹ thuật chính (Key Techniques)

*   **Quản lý IP động & Multi-Domain:** Script `bm.sh` hỗ trợ module xuất IP cho nhiều tên miền khác nhau, giúp giảm thiểu rủi ro bị đưa vào danh sách đen (Blacklist).
*   **Cơ chế chống tấn công (Security):**
    *   Tích hợp **Fail2Ban** để theo dõi log và chặn các IP có dấu hiệu tấn công dò mật khẩu (Brute-force) vào Core API và RoundCube.
    *   Kỹ thuật ngăn chặn **Directory Traversal** (duyệt thư mục trái phép) vừa được cập nhật gần đây.
    *   Tự động quản lý và gia hạn SSL thông qua **ACME/Let's Encrypt**.
*   **Quản lý hạn ngạch (Quota Management):** Hệ thống giám sát dung lượng hòm thư và đưa ra cảnh báo khi sắp hết bộ nhớ.
*   **Email Marketing nâng cao:**
    *   Hỗ trợ **Spintax** (xoay vòng nội dung) để tránh bộ lọc spam.
    *   Theo dõi (Tracking) tỷ lệ mở, tỷ lệ click thông qua các endpoint chuyển hướng.
    *   Xử lý tự động hủy đăng ký (Unsubscribe) và khiếu nại (Complaints).

## 4. Tóm tắt Luồng hoạt động (Workflow)

Hệ thống hoạt động theo một chu trình khép kín:

### Bước 1: Khởi tạo (Cài đặt & Cấu hình)
*   Người dùng chạy `install.sh`. Script này thiết lập môi trường Docker, khởi tạo database PostgreSQL (`init.sql`) và tạo các chứng chỉ SSL tự ký ban đầu.
*   Quản trị viên cấu hình tên miền (DNS records: MX, SPF, DKIM, DMARC) thông qua giao diện quản trị.

### Bước 2: Quản lý Tài nguyên
*   **Mailbox:** Tạo và quản lý hòm thư cá nhân.
*   **Contacts:** Nhập danh sách người nhận (CSV/TXT), phân loại theo Tag hoặc Group.
*   **Templates:** Thiết kế email bằng trình soạn thảo trực quan hoặc AI.

### Bước 3: Thực thi Chiến dịch (Campaign Execution)
*   Hệ thống GoCore lấy danh sách người nhận từ DB -> Xử lý nội dung (Render template, Spintax) -> Đẩy vào hàng đợi Redis.
*   Worker trong GoCore lấy thư từ Redis -> Chuyển qua Postfix để gửi đi.
*   Postfix sử dụng Rspamd để ký DKIM, đảm bảo độ tin cậy của email.

### Bước 4: Giám sát & Phản hồi
*   **Nhận thư:** Postfix nhận thư -> Rspamd kiểm tra spam -> Dovecot lưu vào thư mục `vmail`. Người dùng xem qua RoundCube.
*   **Thống kê:** Hệ thống ghi nhận log gửi thành công/thất bại, lượt mở email và hiển thị biểu đồ trên Overview Dashboard.
*   **Bảo trì:** Các script `rotate_log.sh` tự động nén và dọn dẹp log định kỳ để tối ưu dung lượng đĩa.

---

**Kết luận:** BillionMail không chỉ là một Mail Server, nó là một hệ sinh thái **Email Marketing tự vận hành**. Điểm mạnh lớn nhất là sự kết hợp giữa hiệu năng cao của **Go**, tính ổn định của **Postfix/Dovecot** và khả năng triển khai cực nhanh qua **Docker**.