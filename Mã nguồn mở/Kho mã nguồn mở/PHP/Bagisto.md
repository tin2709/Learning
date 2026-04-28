Dựa trên các tệp tin cấu hình và cấu trúc thư mục từ repository của **Bagisto**, đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng thương mại điện tử mã nguồn mở này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Bagisto là một hệ sinh thái PHP hiện đại, tận dụng tối đa sức mạnh của Laravel:

*   **Framework chính:** **Laravel 12** (phiên bản mới nhất). Điều này cho thấy dự án luôn cập nhật các tính năng bảo mật và hiệu suất mới nhất của PHP.
*   **Ngôn ngữ lập trình:** **PHP 8.3+** (Yêu cầu nghiêm ngặt trong `UPGRADE.md`).
*   **Frontend:**
    *   **Vue.js 3:** Sử dụng cho các thành phần tương tác trong Admin và Shop.
    *   **Vite:** Thay thế Webpack để build tài nguyên nhanh hơn (Hot Module Replacement - HMR).
    *   **Tailwind CSS:** Framework CSS chủ đạo cho giao diện hiện đại, dễ tùy chỉnh.
*   **Cơ sở dữ liệu:** **MySQL 8.0** là mặc định. Ngoài ra còn có sự hỗ trợ của **Redis** cho caching/session và **Elasticsearch 7.17** cho tìm kiếm nâng cao.
*   **AI Integration:** Sử dụng **Laravel AI SDK** (Magic AI) hỗ trợ nhiều nhà cung cấp như OpenAI, Anthropic, Gemini, và đặc biệt là **Ollama** (AI chạy local).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Bagisto mang tính **Modular (Mô-đun hóa)** cực cao, khác biệt với các ứng dụng Laravel thông thường:

*   **Hệ thống Package-Based (Webkul Packages):** Thay vì viết code trong thư mục `app/`, toàn bộ logic cốt lõi nằm trong `packages/Webkul/`. Mỗi tính năng (Sitemap, Shop, Admin, Product, Sales, Tax...) là một package độc lập có đủ: Routes, Controllers, Models, Migrations và Views.
*   **Konekt Concord:** Đây là "xương sống" kiến trúc của Bagisto. Nó cho phép người dùng ghi đè (override) các Model, Repository hoặc Controller của lõi hệ thống mà không cần chỉnh sửa trực tiếp vào mã nguồn gốc.
*   **Repository Pattern:** Bagisto sử dụng `Prettus L5 Repository`. Mọi truy vấn DB không gọi trực tiếp từ Eloquent Model trong Controller mà thông qua các lớp Repository, giúp dễ dàng viết Unit Test và bảo trì.
*   **EAV (Entity-Attribute-Value):** Một kiến trúc database phức tạp dành cho sản phẩm. Nó cho phép admin tạo vô số thuộc tính (màu sắc, kích cỡ, chất liệu...) cho sản phẩm mà không cần thay đổi cấu trúc bảng trong SQL.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Model Proxies:** Kỹ thuật đặc trưng của Concord. Bạn sẽ thấy các class như `ProductProxy`. Điều này cho phép hệ thống gọi Model mà không cần quan tâm Model đó đã được lập trình viên tùy biến hay chưa.
*   **DataGrids:** Một kỹ thuật tự xây dựng để render các bảng dữ liệu lớn trong trang quản trị. Nó xử lý lọc, sắp xếp, và xuất dữ liệu (CSV/XLS) một cách trừu tượng (abstract).
*   **Theme Management:** Hệ thống theme tách biệt giữa Admin và Shop, hỗ trợ fallback (nếu theme tùy chỉnh thiếu file, nó sẽ tự tìm trong theme mặc định).
*   **Event-Driven:** Sử dụng Laravel Events và Listeners để xử lý các tác vụ như gửi email sau khi đặt hàng, cập nhật kho, hoặc thông báo GDPR.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý một yêu cầu điển hình (ví dụ: Xem sản phẩm):

1.  **Bootstrapping:** Yêu cầu đi qua `public/index.php` -> `bootstrap/app.php`. Tại đây, các Service Provider của từng package Webkul sẽ được đăng ký.
2.  **Middleware:** Các lớp bảo vệ kiểm tra: Locale (ngôn ngữ), Currency (tiền tệ), Theme (giao diện đang dùng) và các chính sách bảo mật (Secure Headers).
3.  **Routing:** Laravel nhận diện yêu cầu thuộc về `Shop` hay `Admin` dựa trên cấu hình trong các package tương ứng.
4.  **Business Logic (Repository):** Controller gọi Repository (ví dụ: `ProductRepository`). Nếu tính năng Elasticsearch được bật, Repository sẽ lấy dữ liệu từ chỉ mục (index) của Elasticsearch thay vì MySQL để tăng tốc.
5.  **Vite Asset Loading:** Blade template sử dụng helper `bagisto_assets()` để load các file JS/CSS đã được Vite biên dịch theo đúng theme đã chọn.
6.  **AI Enhancement (Opcional):** Nếu người dùng sử dụng tìm kiếm bằng hình ảnh, hệ thống sẽ gửi dữ liệu qua Magic AI để phân tích vector hình ảnh trước khi truy vấn sản phẩm.

### Tổng kết
Bagisto không chỉ là một trang web bán hàng, nó là một **E-commerce Framework**. Kiến trúc của nó hướng tới các nhà phát triển (Developers) để họ có thể xây dựng thêm các module (Marketplace, POS, B2B) trên nền tảng có sẵn mà không làm hỏng lõi hệ thống. Việc nâng cấp lên **Laravel 12** và tích hợp **AI SDK** cho thấy Bagisto đang định hướng trở thành nền tảng thương mại điện tử hiện đại nhất trong hệ sinh thái PHP.