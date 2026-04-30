Dưới đây là phân tích chi tiết về dự án mã nguồn mở **LimeSurvey**, nền tảng khảo sát trực tuyến phổ biến nhất thế giới, dựa trên cấu trúc thư mục và nội dung tệp tin bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Backend chính:** Sử dụng ngôn ngữ **PHP** (yêu cầu ≥ 7.4).
*   **Framework:** Dựa trên nền tảng **Yii Framework phiên bản 1.1** (một legacy framework nhưng cực kỳ mạnh mẽ và ổn định cho các hệ thống lớn). Điều này được xác nhận qua tệp `composer.json` và cách khởi tạo ứng dụng trong `index.php`.
*   **Frontend:**
    *   **Vue.js:** Đang trong quá trình hiện đại hóa giao diện (đặc biệt là các bảng điều khiển admin) bằng Vue.js. Có tệp `buildVueComponents.js` riêng để quản lý việc đóng gói các thành phần Vue.
    *   **Twig:** Công cụ chuyển đổi giao diện (Template Engine) chính cho các bản khảo sát và email.
    *   **Công cụ Build:** Sử dụng **Yarn/NPM** để quản lý thư viện JS, **Gulp** để transpile SCSS/JS, và **Babel** để tương thích trình duyệt cũ.
*   **Cơ sở dữ liệu:** Hỗ trợ đa dạng qua PDO: MySQL (8.0+), MariaDB, PostgreSQL, và MSSQL.
*   **API:** Cung cấp cả **RemoteControl API** (JSON-RPC/XML-RPC) cũ và **REST API v1** hiện đại (đang phát triển mạnh).

### 2. Tư duy Kiến trúc (Architectural Thinking)

LimeSurvey sử dụng kiến trúc **MVC (Model-View-Controller)** truyền thống của Yii nhưng được tùy biến sâu để phục vụ tính linh hoạt cực cao của khảo sát:

*   **Tính mở rộng (Extensibility):** Kiến trúc plugin rất mạnh mẽ (thư mục `application/core/plugins/`). Hệ thống cho phép can thiệp vào hầu hết các sự kiện (events) của vòng đời khảo sát mà không cần sửa code lõi.
*   **Phân tách câu hỏi (Atomic Question Types):** Mỗi loại câu hỏi (Question Type) được coi là một thực thể riêng biệt với logic render và xử lý dữ liệu riêng (thư mục `application/core/QuestionTypes/`).
*   **Cấu trúc Đa ngôn ngữ (Localization-First):** Tư duy quốc tế hóa được nhúng sâu với hệ thống `gettext` (.mo/.po files) và cơ sở dữ liệu hỗ trợ lưu trữ nội dung theo từng ngôn ngữ (bảng `_l10n`).
*   **Hệ thống Theme phân cấp:** Sử dụng cơ chế kế thừa theme (Inheritance). Người dùng có thể tạo theme tùy chỉnh dựa trên theme gốc (`vanilla`, `fruity`) và chỉ cần ghi đè các tệp cần thiết.

### 3. Các kỹ thuật chính (Key Techniques)

*   **ExpressionScript (EM):** Đây là "bộ não" của LimeSurvey. Một công cụ phân tích cú pháp biểu thức cực kỳ phức tạp (thư mục `application/helpers/expressions/`), cho phép tạo logic rẽ nhánh, tính toán toán học và thay thế dữ liệu động ngay trong thời gian thực khi người dùng trả lời.
*   **Database Schema Abstraction:** Kỹ thuật trừu tượng hóa DB (thư mục `application/core/db/`) giúp ứng dụng chạy được trên nhiều hệ quản trị cơ sở dữ liệu khác nhau mà không thay đổi logic nghiệp vụ.
*   **Asset Management:** Sử dụng `LSYii_AssetManager` để quản lý các tệp tĩnh (CSS/JS). Kỹ thuật này giúp cache-busting (thêm version vào file) và kết hợp nhiều file để tối ưu tốc độ tải trang.
*   **Sodium Encryption:** Sử dụng thư viện `LSSodium` để mã hóa các dữ liệu nhạy cảm của người tham gia khảo sát, đảm bảo tuân thủ GDPR.
*   **Phân tích cấu trúc khảo sát qua XML:** LimeSurvey có khả năng xuất/nhập toàn bộ cấu trúc khảo sát (câu hỏi, logic, theme) qua tệp XML (.lss), giúp di chuyển khảo sát giữa các máy chủ dễ dàng.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi động (Bootstrapping):**
    *   Người dùng truy cập `index.php`.
    *   Hệ thống nạp cấu hình (`application/config/internal.php` và `config.php`).
    *   Yii Framework được khởi tạo và định tuyến (Routing) dựa trên tệp `routes.php`.

2.  **Quản trị (Admin Side):**
    *   `AdminController` quản lý việc tạo khảo sát.
    *   Khi người dùng thiết kế câu hỏi, hệ thống lưu vào các bảng Metadata (`questions`, `groups`, `answers`).
    *   Các assets (Vue components) được nạp để cung cấp trải nghiệm kéo thả mượt mà.

3.  **Thực thi khảo sát (Survey Runtime):**
    *   Khi người trả lời truy cập link khảo sát, `SurveyController` sẽ kích hoạt.
    *   **ExpressionScript** kiểm tra điều kiện hiển thị của từng câu hỏi.
    *   **Twig Renderer** nạp theme và render mã HTML từ các tệp `.twig`.
    *   Dữ liệu câu trả lời được lưu vào bảng động (mỗi khảo sát có một bảng phản hồi riêng `survey_XXXXX` để đảm bảo hiệu suất khi dữ liệu lớn).

4.  **Xử lý kết quả:**
    *   `ExportSurveyResultsService` xử lý việc xuất dữ liệu ra Excel, CSV, PDF, hoặc SPSS thông qua các thư viện trung gian như TCPDF hoặc PhpXlsxWriter.

---
**Kết luận:** LimeSurvey là một dự án có sự kết hợp giữa **di sản (Legacy)** vững chắc của Yii1 và các **công nghệ hiện đại** (Vue.js, REST API). Điểm mạnh nhất của nó không phải là code frontend đẹp mắt mà là logic xử lý khảo sát (ExpressionScript) và khả năng tùy biến sâu rộng qua Plugin/Theme.