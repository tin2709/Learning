Dựa trên mã nguồn và cấu trúc thư mục của dự án **Fail2Ban**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)
Fail2Ban là một khung làm việc (framework) ngăn chặn xâm nhập được viết chủ yếu bằng **Python**:
*   **Ngôn ngữ chính:** **Python (3.5+)**. Dự án tận dụng tính đa nền tảng của Python để chạy trên Linux, BSD, macOS và Solaris.
*   **Xử lý văn bản:** **Regular Expressions (Regex)** là "trái tim" của hệ thống. Nó sử dụng thư viện `re` của Python để phân tích các dòng log phức tạp.
*   **Tương tác hệ thống (Firewall):** Hệ thống không tự chặn kết nối mà tương tác với các công cụ tường lửa có sẵn của hệ điều hành như `iptables`, `nftables`, `pf`, `ipfw`, `firewalld`, và `ufw` thông qua các lệnh shell.
*   **Giám sát file log:** Sử dụng **Gamin**, **PyInotify** (dựa trên sự kiện hạt nhân Linux) hoặc kỹ thuật **Polling** (truy vấn định kỳ) để phát hiện thay đổi trong tệp log.
*   **Cơ sở dữ liệu:** **SQLite** được sử dụng để lưu trữ trạng thái bền vững (persistent data), cho phép hệ thống khôi phục các lệnh cấm (ban) ngay cả sau khi khởi động lại dịch vụ.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Fail2Ban đi theo mô hình **Client-Server** và thiết kế dựa trên các thực thể độc lập:

*   **Mô hình Client-Server:** 
    *   `fail2ban-server`: Một daemon chạy ngầm, chịu trách nhiệm giám sát log và thực thi lệnh cấm.
    *   `fail2ban-client`: Công cụ giao tiếp với server qua **Unix Sockets** để thay đổi cấu hình hoặc kiểm tra trạng thái mà không cần khởi động lại daemon.
*   **Kiến trúc "Jail" (Nhà tù):** Đây là đơn vị quản lý cao nhất. Một Jail là sự kết hợp giữa:
    *   **Filter (Bộ lọc):** Xác định "thế nào là một lần đăng nhập lỗi" dựa trên regex.
    *   **Action (Hành động):** Xác định "phải làm gì khi đạt ngưỡng lỗi" (ví dụ: chặn IP, gửi email, cập nhật DB).
*   **Đa luồng (Multi-threading):** Mỗi Jail chạy trên một luồng (thread) riêng biệt (`JailThread`). Điều này đảm bảo rằng việc giám sát log của dịch vụ này (ví dụ: SSH) không bị ảnh hưởng bởi sự chậm trễ của dịch vụ khác (ví dụ: Apache).

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)
*   **Tính kế thừa và Đa hình (OOP):** 
    *   Dự án định nghĩa các lớp cơ sở như `ActionBase`, `Filter`, `FileFilter`. Các hành động cụ thể (như chặn bằng `iptables` hay gửi mail qua `smtp`) sẽ kế thừa và triển khai lại các phương thức `start`, `stop`, `ban`, `unban`.
*   **Quản lý trạng thái (State Management):**
    *   `FailManager`: Theo dõi số lần thất bại của từng IP trong khoảng thời gian `findtime`.
    *   `BanManager`: Quản lý danh sách các IP đang bị cấm và tính toán thời gian hết hạn (`bantime`).
*   **Cấu hình phân cấp (Hierarchical Configuration):**
    *   Sử dụng định dạng tệp `.conf`. Kỹ thuật quan trọng nhất là cho phép tệp `.local` ghi đè lên `.conf`. Điều này giúp người dùng tùy chỉnh hệ thống mà không làm hỏng mã nguồn gốc của nhà phát triển.
*   **Xử lý sai số thời gian:** Hệ thống có lớp `DateDetector` cực kỳ tinh vi để đoán định dạng ngày tháng trong log (vốn rất không nhất quán giữa các phần mềm).

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng xử lý một địa chỉ IP có hành vi xấu diễn ra như sau:

1.  **Giai đoạn Giám sát (Monitoring):** `Filter` theo dõi tệp tin log được chỉ định. Khi có dòng mới, nó chuyển dòng đó qua các `failregex`.
2.  **Giai đoạn Định danh (Identification):** Nếu một dòng log khớp với regex, `FailManager` sẽ trích xuất địa chỉ IP và thời gian, sau đó tăng biến đếm số lần vi phạm của IP đó.
3.  **Giai đoạn Kiểm tra ngưỡng (Threshold Check):** Hệ thống so sánh số lần vi phạm với `maxretry`. Nếu số lần vi phạm vượt ngưỡng trong khoảng thời gian `findtime`, IP đó sẽ bị chuyển trạng thái sang "To Ban" (Cần bị cấm).
4.  **Giai đoạn Thực thi (Banning):** `Jail` gọi các `Action` tương ứng. Một lệnh shell sẽ được gửi đến tường lửa (ví dụ: `iptables -I INPUT -s <IP> -j DROP`). Thông tin này cũng được ghi vào SQLite.
5.  **Giai đoạn Giải phóng (Unbanning):** Sau khi hết thời gian `bantime`, một luồng kiểm tra định kỳ sẽ kích hoạt lệnh gỡ bỏ cấm (unban) để trả lại quyền truy cập cho IP đó.

### Tổng kết
Kiến trúc của Fail2Ban rất linh hoạt và mạnh mẽ nhờ việc **tách biệt hoàn toàn giữa việc phát hiện lỗi (Filter) và việc xử lý lỗi (Action)**. Nó biến các tệp log tĩnh thành một hệ thống phản ứng động, giúp bảo vệ máy chủ khỏi các cuộc tấn công Brute-force một cách tự động và hiệu quả.