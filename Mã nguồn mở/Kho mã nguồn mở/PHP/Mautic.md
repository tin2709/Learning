Dựa trên mã nguồn và tài liệu kỹ thuật của **Mautic** (phiên bản mới nhất hướng tới v7.0), dưới đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng Marketing Automation mã nguồn mở hàng đầu này:

### 1. Công nghệ cốt lõi (Core Technology)

Mautic là một ứng dụng PHP quy mô doanh nghiệp, sử dụng những công nghệ mạnh mẽ nhất trong hệ sinh thái Symfony:

*   **Framework chính:** **Symfony 7.x** (vừa nâng cấp từ các bản cũ). Đây là sự thay đổi cực lớn, biến Mautic thành một trong những dự án PHP hiện đại nhất hiện nay.
*   **Ngôn ngữ:** **PHP 8.2+**. Yêu cầu strict types, tận dụng các tính năng mới như Constructor Promotion, Readonly properties.
*   **Cơ sở dữ liệu:** **MySQL 8.4+** hoặc **MariaDB 10.11+**. Sử dụng **Doctrine ORM** để quản lý thực thể và **Doctrine Migrations** để quản lý sơ đồ DB.
*   **Xử lý hàng đợi (Queuing):** **Symfony Messenger**. Thay thế hệ thống cũ, cho phép xử lý bất đồng bộ các tác vụ nặng như gửi email hàng loạt, cập nhật segment qua các Transport như Doctrine, Redis, hoặc AMQP.
*   **Gửi Email:** **Symfony Mailer**. Thay thế SwiftMailer đã lỗi thời, hỗ trợ cấu hình DSN linh hoạt cho mọi nhà cung cấp (Sendgrid, Mailgun, Amazon SES...).
*   **API:** Đang chuyển mình sang **API Platform**, giúp tự động hóa việc tạo ra các endpoint REST/GraphQL chuẩn chỉnh.
*   **Frontend:** Chuyển đổi hoàn toàn sang **Twig Template**, sử dụng **Webpack/Grunt** để quản lý Assets và **CKEditor 5** cho các bộ soạn thảo nội dung.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Mautic được thiết kế theo kiểu **Modular Monolith** (Khối thống nhất chia module) với tính mở rộng cực cao:

*   **Kiến trúc Bundle-based:** Mọi tính năng cốt lõi đều là một "Bundle" (ví dụ: `LeadBundle` quản lý contact, `CampaignBundle` quản lý chiến dịch). Mỗi Bundle có đầy đủ Controller, Entity, Event, Service riêng.
*   **Hệ thống Plugin:** Mautic cung cấp một framework cho phép bên thứ ba viết thêm tính năng vào thư mục `plugins/` mà không cần sửa code lõi. Các plugin này có thể lắng nghe các sự kiện hệ thống để can thiệp vào logic.
*   **Sự kiện là trung tâm (Event-Driven):** Tư duy "lỏng" (decoupled) được thể hiện qua việc sử dụng **Symfony Event Dispatcher**. Khi một contact được tạo, một sự kiện `LeadEvents::LEAD_POST_SAVE` được bắn ra, và hàng chục subscriber khác có thể nhận để tính điểm (Point), gửi email chào mừng, hoặc đẩy dữ liệu sang CRM.
*   **Dependency Injection (DI):** Áp dụng triệt để Autowiring và Autoconfiguration. Các service được quản lý chặt chẽ qua file `services.php`, giúp việc thay thế hoặc mở rộng logic trở nên dễ dàng.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Abstract Model Layer:** Mautic sử dụng một lớp trung gian `AbstractCommonModel` bên trên Doctrine Repositories. Model này chứa logic nghiệp vụ cấp cao (như validate dữ liệu trước khi lưu), trong khi Repository chỉ lo việc truy vấn SQL.
*   **Command Pattern cho CLI:** Vì Mautic xử lý hàng triệu contact, các tác vụ quan trọng (cập nhật Segment, kích hoạt Campaign) được thực hiện qua **Symfony Console Commands**. Điều này cho phép chạy ngầm qua Cronjob để đảm bảo hiệu suất web.
*   **Trait-based Logic:** Sử dụng Trait rất nhiều để tái sử dụng code giữa các Entity (ví dụ: `TranslationEntityTrait` cho các thực thể cần đa ngôn ngữ, `VariantEntityTrait` cho A/B testing).
*   **Generated Columns:** Sử dụng kỹ thuật lưu trữ dữ liệu tính toán sẵn trong Database (Generated Columns) để tăng tốc độ tìm kiếm contact dựa trên các trường tùy chỉnh.
*   **Cơ chế Locking:** Sử dụng `Symfony Lock` để đảm bảo khi chạy các lệnh CLI song song, một contact không bị xử lý bởi hai tiến trình cùng lúc, tránh xung đột dữ liệu.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của dữ liệu trong Mautic rất phức tạp nhưng logic:

1.  **Tiếp nhận dữ liệu (Ingestion):**
    *   Tracking Pixel (JS) trên website khách hàng gửi yêu cầu về `PageBundle`.
    *   Dữ liệu được chuyển qua `LeadBundle` để xác định contact (qua IP, Cookie hoặc Email).
2.  **Phân đoạn (Segmentation):**
    *   Lệnh CLI `mautic:segments:update` quét qua toàn bộ contact.
    *   Sử dụng Query Builder phức tạp để lọc những contact thỏa mãn điều kiện và đưa vào bảng `lead_lists_leads`.
3.  **Chiến dịch (Campaign Execution):**
    *   Lệnh CLI `mautic:campaigns:trigger` tìm kiếm các "Event" cần xử lý trong chiến dịch.
    *   Dựa trên các "Decision" (Contact có click link không?) hoặc "Action" (Gửi email), hệ thống đẩy các task vào **Symfony Messenger**.
4.  **Liên lạc (Communication):**
    *   Messenger Worker lấy task từ hàng đợi.
    *   Gọi `Symfony Mailer` hoặc các bộ tích hợp SMS.
    *   Ghi nhật ký (Audit Log) và cập nhật thống kê (Stats) vào các bảng riêng biệt để báo cáo.

**Tổng kết:** Mautic là một kỳ quan về kỹ thuật PHP Symfony. Nó không dành cho những người thích sự đơn giản, mà dành cho các hệ thống đòi hỏi **sự tùy biến vô hạn** và **Digital Sovereignty** (quyền sở hữu dữ liệu tuyệt đối). Việc nâng cấp lên Symfony 7 và PHP 8.2 cho thấy Mautic đang chuẩn bị cho một kỷ nguyên hiệu suất cao hơn và ổn định hơn.