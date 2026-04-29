Dựa trên phân tích mã nguồn và tài liệu kiến trúc của **Basecamp Fizzy**, đây là báo cáo chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Technologies)

Fizzy là hình mẫu tiêu biểu cho triết lý "The Majestic Monolith" của 37signals:

*   **Backend:** **Ruby on Rails** (phiên bản mới nhất từ nhánh `main`), chạy trên **Ruby 3.4.8**.
*   **Frontend (Hotwire Stack):** Sử dụng **Turbo** (bao gồm Turbo Streams cho cập nhật thời gian thực) và **Stimulus JS**. Fizzy không sử dụng các framework SPA (như React/Vue) mà dùng **Importmaps** để quản lý JavaScript và **Propshaft** cho tài sản (assets).
*   **Hệ sinh thái "Solid":** Một điểm đặc biệt là dự án đã loại bỏ hoàn toàn Redis để giảm chi phí hạ tầng, thay thế bằng các giải pháp dựa trên Database:
    *   **Solid Queue:** Xử lý tác vụ nền (Background Jobs).
    *   **Solid Cache:** Bộ nhớ đệm.
    *   **Solid Cable:** Xử lý WebSockets.
*   **Cơ sở dữ liệu:** **SQLite** cho môi trường phát triển và **MySQL (Trilogy adapter)** cho môi trường production.
*   **Deployment:** Sử dụng **Kamal** để deploy trực tiếp lên máy chủ ảo và **Thruster** làm HTTP/2 proxy/cân bằng tải.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Fizzy tập trung vào việc tối giản hóa vận hành nhưng vẫn đảm bảo khả năng mở rộng:

*   **Multi-Tenancy dựa trên URL:** Thay vì dùng subdomain (như `tenant.fizzy.com`), Fizzy dùng tiền tố đường dẫn (ví dụ: `fizzy.com/12345/boards`). Một Middleware (`AccountSlug::Extractor`) sẽ tách ID tài khoản từ URL và gán vào `Current.account`, giúp logic code bên trong hoàn toàn không bị ảnh hưởng bởi cấu trúc đa người dùng.
*   **Tách biệt Identity và User:** 
    *   `Identity`: Đại diện cho thực thể người dùng toàn cầu (Email).
    *   `User`: Đại diện cho tư cách thành viên của một Identity trong một Account cụ thể. Một người có thể thuộc nhiều công ty (Account) khác nhau.
*   **Vanilla Rails:** Tuân thủ chặt chẽ việc giữ Controller mỏng và đưa logic nghiệp vụ vào Model. Fizzy từ chối sử dụng "Service Objects" hay "Form Objects" trừ khi cực kỳ cần thiết.
*   **Sharded Full-Text Search:** Thay vì dùng Elasticsearch/Algolia, Fizzy tự xây dựng hệ thống tìm kiếm toàn văn trên MySQL với 16 shard, chia dữ liệu dựa trên mã hash của Account ID để đảm bảo hiệu suất cực cao.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Hệ thống Entropy (Chống "rác" công việc):** Đây là tính năng cốt lõi. Các thẻ (Card) sẽ tự động bị đẩy vào trạng thái "Not Now" (postponed) nếu không có hoạt động trong một khoảng thời gian nhất định (mặc định do Account hoặc Board quy định). Điều này giúp bảng Kanban luôn gọn gàng.
*   **UUIDv7 Primary Keys:** Tất cả các bảng sử dụng UUIDv7 dưới dạng mã hóa base36 (chuỗi 25 ký tự). Điều này cho phép sắp xếp bản ghi theo thời gian thực thi (mặc dù là UUID) và giúp việc tạo dữ liệu mẫu (fixtures) trở nên deterministic hơn trong quá trình test.
*   **Polymorphic Events:** Mọi hành động quan trọng (tạo card, comment, gán người) đều tạo ra một bản ghi trong bảng `Event`. Bảng này sau đó điều hướng luồng thông báo (Notifications), Webhooks và dòng thời gian hoạt động (Activity Timeline).
*   **View Transitions:** Tận dụng API View Transitions của trình duyệt kết hợp với Turbo để tạo ra các hiệu ứng chuyển trang mượt mà như ứng dụng di động.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Tiếp nhận Request:** Request đến -> Middleware trích xuất Account ID từ URL -> Thiết lập ngữ cảnh bảo mật và dữ liệu trong `Current`.
2.  **Xác thực:** Fizzy ưu tiên **Magic Link** (đăng nhập không mật khẩu qua email) và **Passkeys**. Identity sau khi xác thực thành công sẽ gán Session vào cookie đã ký (signed cookie).
3.  **Xử lý Kanban:** Người dùng kéo thả thẻ Card -> Stimulus gửi yêu cầu tới Controller -> Model cập nhật vị trí -> Phát tín hiệu (Broadcast) qua Turbo Stream tới tất cả các thành viên khác đang xem bảng đó.
4.  **Tác vụ nền:** Khi có sự kiện (Event), hệ thống đẩy các việc như gửi Email thông báo hay Dispatch Webhook vào **Solid Queue**. Các Worker sẽ lấy việc từ chính bảng database để xử lý.
5.  **Dọn dẹp tự động:** Một Job định kỳ (cron-like via Solid Queue) quét các Card "nguội" để thực hiện quy trình auto-postpone (Entropy), duy trì sự tập trung cho đội ngũ.

**Kết luận:** Fizzy là một minh chứng cho thấy một ứng dụng SaaS hiện đại, quy mô lớn có thể được xây dựng vô cùng tinh gọn bằng cách tận dụng tối đa sức mạnh của database quan hệ và các công cụ "mặc định" của Rails.