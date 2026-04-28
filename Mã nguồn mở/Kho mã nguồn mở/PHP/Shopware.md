Dựa trên cấu trúc thư mục, các bản ghi quyết định kiến trúc (ADR) và tệp cấu hình của Shopware 6 mà bạn cung cấp, dưới đây là phân tích chuyên sâu về hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)

Shopware 6 được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào tính linh hoạt và khả năng mở rộng:

*   **Backend:** Sử dụng **Symfony 7** (phiên bản mới nhất) làm nền tảng framework. Ngôn ngữ lập trình yêu cầu **PHP 8.2+**. 
*   **Frontend Administration:** Chuyển đổi từ Vue 2 sang **Vue.js 3**, kết hợp với **Pinia** (thay thế Vuex) để quản lý trạng thái và **Vite** để build công cụ, giúp tăng tốc độ phát triển.
*   **Frontend Storefront:** Sử dụng **Twig** làm công cụ render template, **Bootstrap 5** cho CSS framework và **Webpack 5/Vite** để xử lý tài nguyên tĩnh.
*   **Cơ sở dữ liệu & Search:** Hỗ trợ **MySQL 8.0+** hoặc **MariaDB 10.11+**. Đặc biệt, hệ thống tích hợp sâu với **OpenSearch 2** (hoặc Elasticsearch 8) để xử lý tìm kiếm và lập chỉ mục dữ liệu lớn.
*   **Cấu trúc dữ liệu:** Sử dụng **UUID v7** thay vì ID tăng dần (Auto-increment) để hỗ trợ tốt hơn cho hệ thống phân tán và đồng bộ hóa dữ liệu.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Shopware 6 tuân theo triết lý **"API-first"** và **"Headless"**:

*   **Tách biệt hoàn toàn (Decoupled):** Nhân hệ thống (Core) hoàn toàn độc lập với giao diện (Storefront/Admin). Mọi hành động trong Admin hoặc ngoài Storefront đều thực hiện qua các API route.
    *   *Admin API:* Dùng cho quản lý CRUD, yêu cầu quyền ACL cao.
    *   *Store API:* Dùng cho khách hàng, tối ưu hóa cho hiệu suất và caching (phù hợp cho PWA/Mobile App).
    *   *Sync API:* Chuyên dụng cho việc nhập/xuất dữ liệu hàng loạt.
*   **Data Abstraction Layer (DAL):** Đây là điểm khác biệt lớn nhất. Shopware không sử dụng Doctrine ORM một cách trực tiếp để truy vấn dữ liệu theo cách truyền thống. Thay vào đó, họ xây dựng một lớp DAL riêng để quản lý thực thể (Entities), cho phép các Plugin mở rộng dữ liệu (Extension/Composition) mà không làm hỏng cấu trúc bảng gốc.
*   **Hệ thống mở rộng kép (App vs. Plugin):**
    *   *Plugin System:* Cho phép can thiệp sâu vào nhân PHP, phù hợp cho cài đặt On-premise.
    *   *App System:* Dựa trên Webhooks và iFrame, không chạy code PHP trên server của Shopware, cho phép mở rộng hệ thống trong môi trường Cloud/SaaS an toàn.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Criteria API:** Thay vì viết SQL thủ công, Shopware sử dụng đối tượng `Criteria` để truy vấn. Điều này cho phép các plugin khác có thể "nghe" các sự kiện tìm kiếm và thêm bộ lọc, sắp xếp hoặc associations vào truy vấn ban đầu một cách linh hoạt.
*   **Decoration Pattern:** Shopware ưu tiên kỹ thuật **Service Decoration** thay vì kế thừa (Inheritance). Điều này cho phép nhiều plugin cùng can thiệp vào một Service mà không gây ra xung đột (chaining decorators).
*   **Event-driven Architecture:** Hệ thống sử dụng Symfony Event Dispatcher rộng rãi. Hầu hết logic nghiệp vụ được thực hiện qua các `EventSubscriber`. Mọi thay đổi dữ liệu đều kích hoạt các sự kiện `EntityWrittenEvent` để xử lý hậu kỳ (như xóa cache, cập nhật index).
*   **Versioning & Drafts:** DAL hỗ trợ cơ chế Versioning. Khi bạn chỉnh sửa một thực thể (ví dụ: Product), hệ thống có thể tạo ra một bản copy (version mới), sau khi chỉnh sửa xong mới "merge" vào bản chính (Live version) để tránh làm gián đoạn hiển thị.

### 4. Luồng hoạt động hệ thống (System Workflow)

*   **Xử lý Request:**
    1.  `RequestTransformer` nhận diện domain, ngôn ngữ, tiền tệ từ URL.
    2.  Hệ thống xác định `SalesChannelContext` (chứa thông tin khách hàng, thuế, quy tắc giá).
    3.  Router chuyển hướng đến Controller tương ứng.
    4.  Controller gọi Store-API hoặc DAL để lấy dữ liệu dưới dạng `Struct` (thực thể đã được format).
*   **Luồng Giỏ hàng (Cart Process):**
    Sử dụng cơ chế **Collector & Processor**. Khi giỏ hàng thay đổi:
    1.  *Collectors:* Thu thập dữ liệu sản phẩm, giá ưu đãi, thông tin vận chuyển.
    2.  *Processors:* Tính toán lại thuế, tổng tiền, áp dụng quy tắc khuyến mãi (Promotion).
    3.  Kết quả được lưu vào Database hoặc Redis.
*   **Luồng Lập chỉ mục (Indexing Workflow):**
    Khi dữ liệu thay đổi trong MySQL, một thông điệp được gửi vào **Message Queue**. Các `EntityIndexer` chạy ngầm sẽ cập nhật dữ liệu này sang OpenSearch và các bảng SEO URL. Điều này giúp giao diện Storefront phản hồi cực nhanh vì không phải thực hiện các câu lệnh JOIN phức tạp trên MySQL.
*   **Flow Builder:** Đây là tầng logic nghiệp vụ tự động. Admin có thể cấu hình luồng công việc (ví dụ: "Nếu đơn hàng đã thanh toán" -> "Thực hiện hành động: Gửi mail" -> "Và tạo file PDF"). Logic này được thực thi qua các `FlowEvents` mà không cần viết code.

### Kết luận
Shopware 6 không chỉ là một mã nguồn mở bán hàng, mà là một **e-commerce framework** mạnh mẽ. Nó ép buộc lập trình viên đi theo các chuẩn mực nghiêm ngặt về mở rộng (Decoration, DAL, Events) để đảm bảo hệ thống có thể nâng cấp liên tục mà không làm hỏng các tùy biến từ bên thứ ba.