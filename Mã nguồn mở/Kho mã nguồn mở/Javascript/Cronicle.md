Dựa trên nội dung các tệp tin bạn cung cấp về dự án **Cronicle**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống này bằng tiếng Việt:

---

### 1. Công nghệ cốt lõi (Core Technologies)
Cronicle được xây dựng trên một hệ sinh thái JavaScript hiện đại, tập trung vào hiệu năng và khả năng mở rộng:

*   **Runtime:** **Node.js** (Khuyến nghị bản LTS).
*   **Hệ sinh thái PixlCore:** Tác giả sử dụng một bộ các module tự viết (prefix `pixl-`) để tối ưu hóa quyền kiểm soát:
    *   `pixl-server`: Khung (framework) chính để xây dựng daemon.
    *   `pixl-server-storage`: Hệ thống lưu trữ trừu tượng (hỗ trợ Local Disk, S3, Couchbase).
    *   `pixl-server-web`: Web server tích hợp.
    *   `pixl-request` & `pixl-mail`: Xử lý HTTP request và gửi email.
*   **Giao tiếp thời gian thực:** **Socket.io** được dùng để đẩy trạng thái công việc (job status) và log trực tiếp từ server lên trình duyệt.
*   **Giao diện người dùng:** HTML5/CSS3 dựa trên khung **pixl-webapp**, sử dụng **jQuery**, **Moment.js** (xử lý thời gian/múi giờ) và **Chart.js** (vẽ biểu đồ hiệu năng).
*   **Bảo mật:** **BCrypt** để băm mật khẩu và hệ thống **API Key** để xác thực các ứng dụng bên thứ ba.

### 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Cronicle được thiết kế theo mô hình **Distributed Primary/Worker** (Chủ - Tớ phân tán):

*   **Tính sẵn sàng cao (High Availability):** Hỗ trợ nhiều server "Primary backup". Nếu Primary server hiện tại chết, các server khác sẽ tự động bầu chọn (với độ ưu tiên theo bảng chữ cái của hostname) để lên thay thế.
*   **Trừu tượng hóa lưu trữ (Storage Abstraction):** Dữ liệu không bị bó buộc vào một cơ sở dữ liệu duy nhất. Người dùng có thể dùng File System (NFS) cho quy mô nhỏ hoặc S3/Couchbase cho quy mô lớn.
*   **Triết lý "Language Agnostic" (Không phụ thuộc ngôn ngữ):** Các Plugin không nhất thiết phải viết bằng Node.js. Miễn là ngôn ngữ đó có thể đọc/ghi định dạng JSON qua STDIN/STDOUT, nó có thể trở thành một Plugin (Shell, Python, PHP, Perl, v.v.).
*   **Cơ chế Cursor-based Scheduling:** Thay vì dùng hàng đợi (queue) truyền thống, Cronicle sử dụng một "con trỏ" thời gian cho mỗi sự kiện. Điều này cho phép hệ thống "đuổi kịp" (Catch-up/Run All Mode) các tác vụ bị lỡ khi server tạm dừng.

### 3. Các kỹ thuật then chốt (Key Techniques)

*   **Giao tiếp qua Pipe (STDIN/STDOUT JSON):** Khi một Job chạy, Cronicle spawn một tiến trình con, đẩy dữ liệu cấu hình dưới dạng JSON vào STDIN của tiến trình đó. Plugin phản hồi trạng thái, tiến độ (%) và dữ liệu thống kê cũng qua JSON ở STDOUT.
*   **Tự động khám phá (Auto-discovery):** Sử dụng **UDP Broadcast** trên cổng 3014 để các server trong cùng mạng LAN tự nhận diện nhau mà không cần cấu hình IP thủ công.
*   **Quản lý tài nguyên (Resource Monitoring):** Theo dõi trực tiếp CPU và Memory của từng Job (bao gồm cả các tiến trình con do Job đó sinh ra) và tự động ngắt (abort) nếu vượt quá ngưỡng cho phép.
*   **Hệ thống móc nối (Chaining & Webhooks):** Cho phép tạo ra các luồng công việc (workflow) phức tạp: Job A thành công -> chạy Job B; hoặc gửi thông báo JSON đến các URL bên ngoài (Slack, Teams) khi Job kết thúc.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của một tác vụ trong Cronicle diễn ra như sau:

1.  **Định nghĩa (Configuration):** Người dùng tạo **Category** (nhóm), **Plugin** (tệp thực thi) và **Event** (lịch trình) thông qua giao diện Web.
2.  **Lập lịch (Scheduling):** Bộ máy lập lịch trên **Primary Server** kiểm tra mỗi phút một lần. Nếu khớp thời gian hoặc có yêu cầu "Run Now", một **Job** sẽ được tạo ra.
3.  **Phân phối (Distribution):** Primary Server dựa trên thuật toán (Round Robin, Least Load, hoặc chỉ định đích) để chọn một **Worker Server** phù hợp.
4.  **Thực thi (Execution):** 
    *   Worker nhận lệnh, tạo một tệp log riêng cho Job.
    *   Worker thực thi Plugin, truyền tham số qua JSON.
    *   Trong khi chạy, Plugin gửi dữ liệu tiến độ (ví dụ: `{"progress": 0.5}`) về Worker.
5.  **Theo dõi (Monitoring):** Worker đẩy dữ liệu thời gian thực và nội dung log về Primary Server qua WebSockets để hiển thị lên UI cho người dùng xem trực tiếp.
6.  **Hoàn tất (Completion):** 
    *   Plugin thoát với mã lỗi (exit code).
    *   Worker tổng hợp kết quả, thời gian chạy, tài nguyên tiêu thụ và báo về Primary.
    *   Primary lưu lịch sử, gửi Email/Webhook thông báo và kích hoạt các Job phụ thuộc (nếu có).

---
**Lưu ý quan trọng từ README:** Tác giả thông báo đã phát triển **xyOps™** như một "người kế nhiệm" tâm linh cho Cronicle, nhưng vẫn sẽ duy trì Cronicle cho các bản vá lỗi và bảo mật.