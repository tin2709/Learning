Dựa trên các tệp tin và cấu trúc thư mục của **XWiki Platform**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Backend (Java Ecosystem):**
    *   **Ngôn ngữ chính:** Java 11/17 (chiếm ~88%).
    *   **Dependency Injection (DI) & Component Manager:** Sử dụng hệ thống component riêng của XWiki (dựa trên JSR-330) thay vì Spring. Các thành phần được định nghĩa qua `@Component` và khai báo trong `META-INF/components.txt`.
    *   **Persistence (ORM):** Hibernate là nền tảng để giao tiếp với các DB (MySQL, PostgreSQL, Oracle, HSQLDB).
    *   **REST API:** Sử dụng Jersey (JAX-RS) và Jackson để cung cấp dịch vụ web.
    *   **Search Engine:** Tích hợp sâu với Apache Solr (cho tìm kiếm toàn văn) và Elasticsearch (cho dữ liệu Active Installs).
    *   **Scripting:** Hỗ trợ đa ngôn ngữ script chạy trực tiếp trên wiki như Groovy, Velocity, và Python.

*   **Frontend & Web:**
    *   **Framework:** Chuyển dịch từ JavaScript thuần sang Vue.js và TypeScript (thấy trong `livedata-webjar` và `blocknote-webjar`).
    *   **Build Tools:** Maven cho Java, PNPM và NX cho các module JavaScript/Node.
    *   **UI Framework:** Flamingo Skin dựa trên Bootstrap và LESS CSS.
    *   **Editor:** CKEditor được tùy biến mạnh mẽ để hỗ trợ cú pháp Wiki.

### 2. Tư duy Kiến trúc (Architectural Thinking)

XWiki được thiết kế theo hướng **Extensible Platform (Nền tảng có thể mở rộng)** với các tư duy chủ đạo:

*   **Micro-kernel & Modularization:** Chia nhỏ hệ thống thành hàng trăm module (`xwiki-platform-core` chứa hàng loạt module nhỏ). Mỗi tính năng (như `mentions`, `ratings`, `notifications`) là một module độc lập.
*   **Kiến trúc dựa trên Component:** Mọi thứ trong XWiki đều là "Component". Điều này cho phép thay thế (override) bất kỳ logic lõi nào mà không cần sửa mã nguồn gốc bằng cách sử dụng `@Named` hoặc độ ưu tiên (priority).
*   **Wiki-on-Wiki (Data-driven UI):** Một phần lớn logic giao diện và nghiệp vụ được lưu trữ dưới dạng các trang Wiki (XML files trong các module `-ui`). Điều này cho phép người dùng sửa đổi ứng dụng ngay trên trình duyệt.
*   **Tách biệt Rendering:** Hệ thống Rendering (`xwiki-rendering`) tách biệt hoàn toàn với logic lưu trữ, cho phép chuyển đổi từ cú pháp Wiki sang HTML, PDF, hoặc Markdown một cách linh hoạt.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Observation Manager (Event-Driven):** Sử dụng `EventListener` để xử lý các tác vụ bất đồng bộ. Ví dụ: Khi XWiki khởi động xong (`ApplicationReadyEvent`), listener sẽ tự động kích hoạt tiến trình gửi "ping" báo cáo trạng thái hệ thống (`ActiveInstallsInitializerListener`).
*   **Script Service:** Cung cấp các lớp "Bridge" (như `ActiveInstallsScriptService`) để người dùng có thể gọi các API Java phức tạp từ các ngôn ngữ script đơn giản như Velocity trong trang Wiki.
*   **Strategy Pattern:** Thấy rõ trong việc xử lý dữ liệu Ping (`PingDataProvider`). Có nhiều provider khác nhau (OS, Java, Database, Memory) cùng thực thi một interface để thu thập dữ liệu.
*   **AOP (Aspect-Oriented Programming):** Sử dụng AspectJ (trong các module `legacy`) để duy trì khả năng tương thích ngược (Backward Compatibility) mà không làm bẩn mã nguồn hiện tại.
*   **Data Migration:** Hệ thống có cơ chế di cư dữ liệu tự động (`DataMigration`) dựa trên phiên bản, giúp nâng cấp cấu trúc DB và dữ liệu khi cập nhật phiên bản phần mềm.

### 4. Luồng hoạt động hệ thống (System Operation Workflow)

1.  **Giai đoạn Bootstrap:**
    *   Servlet Container (Tomcat/Jetty) tải XWiki.
    *   `XWikiServletContextListener` khởi tạo `ComponentManager`.
    *   Tất cả các `Initializable` component được nạp.
2.  **Xử lý yêu cầu (Request Handling):**
    *   Yêu cầu HTTP đi qua `RoutingFilter`.
    *   Nếu là API, Jersey REST sẽ tiếp nhận. Nếu là trang Wiki, hệ thống sẽ tải nội dung từ Database (thông qua Hibernate).
    *   **Rendering Pipeline:** Nội dung trang Wiki (dạng cú pháp XWiki) -> Parser -> XDOM (Abstract Syntax Tree) -> Transformations (Xử lý macro, icon) -> Renderer -> HTML.
3.  **Hệ thống thông báo & Luồng sự kiện:**
    *   Người dùng thực hiện hành động (ví dụ: Lưu trang).
    *   `ObservationManager` phát tín hiệu `DocumentUpdatedEvent`.
    *   Các Listener liên quan (như `SolrIndexEventListener`) sẽ nhận sự kiện để cập nhật index tìm kiếm hoặc gửi email thông báo bất đồng bộ.
4.  **Hệ thống việc làm (Job System):**
    *   Các tác vụ nặng (như Import XAR, Export PDF, Install Extension) được đưa vào `JobRunner` để chạy ngầm, tránh làm treo giao diện người dùng.

### Tổng kết
XWiki không chỉ là một ứng dụng mà là một **Application Server cho Wiki**. Kiến trúc của nó cực kỳ phức tạp và linh hoạt, ưu tiên khả năng mở rộng thông qua hệ thống Component và khả năng lập trình hóa ngay trên nội dung dữ liệu (Scripting in pages). Đây là một mẫu hình tiêu biểu cho các hệ thống Enterprise CMS lớn.