Dựa trên mã nguồn và tài liệu hệ thống của **Vendure**, đây là phân tích chi tiết về dự án thương mại điện tử chuyên nghiệp này:

### 1. Công nghệ cốt lõi (Core Technologies)
Vendure được xây dựng trên một ngăn xếp công nghệ (stack) hiện đại, tập trung vào hiệu suất và khả năng mở rộng:
*   **Ngôn ngữ:** **TypeScript (94.5%)** là ngôn ngữ chủ đạo, đảm bảo tính an toàn về kiểu dữ liệu (type-safety) cho toàn bộ hệ thống.
*   **Backend Framework:** **NestJS** (Dựa trên Node.js). Cung cấp cấu trúc Dependency Injection mạnh mẽ và kiến trúc mô-đun.
*   **API Layer:** **GraphQL**. Sử dụng Apollo Server để cung cấp API cho cả Shop (khách hàng) và Admin (quản trị), cho phép client chỉ lấy đúng dữ liệu cần thiết.
*   **ORM (Object-Relational Mapping):** **TypeORM**. Hỗ trợ nhiều loại DB như MySQL, MariaDB, PostgreSQL, và SQLite.
*   **Quản lý Monorepo:** **Lerna** kết hợp với **Bun** (vừa mới chuyển từ npm sang Bun để tối ưu tốc độ cài đặt và bảo mật).
*   **Frontend (Admin UI):** Sử dụng **React** (cho dashboard mới) và **Angular** (cho phiên bản admin-ui cũ), đi kèm với **Vite** để build nhanh.
*   **Hàng đợi công việc (Job Queue):** **BullMQ** (dựa trên Redis) để xử lý các tác vụ nền nặng nề.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Vendure được thiết kế theo hướng **"Headless"** và **"Extensible"**:
*   **API-First / Headless:** Tách biệt hoàn toàn backend logic khỏi giao diện người dùng. Backend chỉ trả về dữ liệu qua GraphQL, cho phép tích hợp với mọi loại frontend (Next.js, Vue, Mobile App, IoT...).
*   **Kiến trúc Plugin (Plugin Architecture):** Mọi tính năng mở rộng đều được đóng gói thành Plugin. Các plugin có thể thêm các trường dữ liệu mới (Custom Fields), các thực thể mới, hoặc thay đổi logic xử lý thông qua cấu hình.
*   **Cơ chế Chiến lược (Strategy Pattern):** Vendure sử dụng các "Strategy" để xử lý các logic nghiệp vụ biến thiên như: `PricingStrategy` (tính giá), `ShippingEligibilityChecker` (kiểm tra vận chuyển), `TaxZoneStrategy` (tính thuế theo vùng).
*   **Kiến trúc Server-Worker:** Hệ thống chia làm 2 tiến trình:
    1.  **Server:** Xử lý các request API trực tiếp.
    2.  **Worker:** Xử lý các tác vụ tốn thời gian (gửi email, đánh chỉ số tìm kiếm, xử lý thanh toán) thông qua hàng đợi công việc.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Finite State Machine (FSM):** Sử dụng máy trạng thái hữu hạn để quản lý vòng đời của **Đơn hàng (Order)** và **Thực hiện đơn hàng (Fulfillment)**. Điều này đảm bảo dữ liệu không bị chuyển trạng thái sai logic (ví dụ: không thể giao hàng nếu chưa thanh toán).
*   **Custom Fields:** Cho phép người dùng thêm các trường dữ liệu vào các thực thể cốt lõi (như Product, Customer, Order) ngay trong file cấu hình mà không cần can thiệp sâu vào code lõi.
*   **Translatable Entities:** Kỹ thuật lưu trữ đa ngôn ngữ bằng cách tách các trường cần dịch (như tên, mô tả) sang một bảng riêng (`Translation`), liên kết với thực thể chính và mã ngôn ngữ.
*   **Channels:** Hỗ trợ đa cửa hàng (multi-tenancy) trên một instance duy nhất. Mỗi "Channel" có thể có đơn vị tiền tệ, ngôn ngữ, kho hàng và danh mục sản phẩm riêng.
*   **Code Generation:** Sử dụng `graphql-code-generator` và `gql.tada` để tự động tạo ra các kiểu dữ liệu TypeScript từ Schema GraphQL, giúp lập trình viên tránh lỗi typo khi làm việc với API.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)
1.  **Khởi tạo Request:** Client (Storefront) gửi một GraphQL request kèm theo `vendure-token` (để xác định Channel) và `languageCode`.
2.  **Xử lý API:** Server nhận request, đi qua các Middleware/Guards để kiểm tra quyền hạn (Permissions) và session.
3.  **Hệ thống Logic:**
    *   Nếu là đặt hàng: Hệ thống sử dụng FSM để chuyển trạng thái từ `AddingItems` sang `ArrangingPayment`.
    *   Các Strategy sẽ được gọi để tính toán giá sản phẩm, thuế và phí vận chuyển dựa trên địa chỉ khách hàng.
4.  **Xử lý tác vụ nền:** Nếu thao tác tạo ra các công việc nặng (ví dụ: hoàn tất thanh toán), Server đẩy một Job vào Redis. Worker sẽ nhận Job này để gửi email xác nhận và cập nhật tồn kho.
5.  **Cập nhật tìm kiếm:** Mọi thay đổi về sản phẩm sẽ kích hoạt một sự kiện (Event Bus), sau đó Plugin tìm kiếm (Elasticsearch hoặc Default SQL) sẽ cập nhật lại chỉ mục (index) để khách hàng tìm thấy sản phẩm mới ngay lập tức.
6.  **Phản hồi:** Server trả về dữ liệu JSON theo đúng định dạng GraphQL mà client yêu cầu.

**Kết luận:** Vendure là một framework cực kỳ linh hoạt, phù hợp cho các doanh nghiệp cần một nền tảng thương mại điện tử tùy biến sâu, hỗ trợ đa kênh và đa quốc gia với trải nghiệm phát triển (DX) tuyệt vời nhờ TypeScript.