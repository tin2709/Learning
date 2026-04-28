Dựa trên mã nguồn và cấu trúc thư mục của **Saleor Core**, đây là phân tích chuyên sâu về công nghệ, kiến trúc và cách vận hành của hệ thống thương mại điện tử headless hàng đầu này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Saleor sử dụng một tập hợp các công nghệ Python hiện đại nhất, tập trung vào hiệu suất cao và khả năng mở rộng:

*   **Ngôn ngữ & Runtime:** Python 3.12, chạy trên nền **Uvicorn (ASGI)**. Việc sử dụng ASGI thay vì WSGI truyền thống cho phép xử lý các kết nối bất đồng bộ (asynchronous) hiệu quả hơn.
*   **Framework chính:** **Django 5.x**. Tuy nhiên, Saleor không dùng hệ thống Template của Django mà chỉ tận dụng ORM, quản lý Migration và hệ thống Middleware.
*   **API Layer:** **GraphQL (Graphene-Django)**. Đây là "linh hồn" của Saleor. Hệ thống chỉ cung cấp API GraphQL, không có REST API truyền thống, giúp tối ưu hóa việc lấy dữ liệu (avoid over-fetching).
*   **Quản lý thư viện:** **uv** (Python package manager mới của Astral). Saleor đã chuyển sang dùng `uv` và `pyproject.toml` để đảm bảo tốc độ cài đặt và quản lý dependency chặt chẽ (qua `uv.lock`).
*   **Database & Search:**
    *   **PostgreSQL:** Là cơ sở dữ liệu chính.
    *   **PostgreSQL Full-text Search:** Saleor sử dụng `SearchVector` và `SearchRank` ngay trong Postgres thay vì các công cụ bên ngoài như Elasticsearch để giảm bớt độ phức tạp của hạ tầng.
*   **Xử lý tác vụ ngầm:** **Celery + Redis/Valkey**. Dùng để gửi email, xử lý các tác vụ migration dữ liệu lớn hoặc đồng bộ hóa kho.
*   **Quan sát (Observability):** Tích hợp sâu **OpenTelemetry** để tracing và **Sentry** để quản lý lỗi.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Saleor được xây dựng theo triết lý **API-first** và **Composable Commerce**:

*   **Headless & Technology-agnostic:** Saleor Core chỉ là một server API. Bạn có thể xây dựng Storefront bằng React, Vue, hay App Mobile tùy ý.
*   **App & Webhook Architecture:** Thay vì cài đặt plugin (thư mục code trực tiếp) như WordPress hay Magento, Saleor mở rộng tính năng thông qua **Saleor Apps**. Các App này giao tiếp với Core qua Webhooks và GraphQL API. Điều này giúp Core luôn "sạch" và dễ dàng nâng cấp.
*   **Native Multi-channel:** Saleor thiết kế sẵn mô hình "Channel". Mỗi Channel có cấu hình riêng về tiền tệ, giá cả, tồn kho và ngôn ngữ, cho phép vận hành thương mại xuyên biên giới ngay từ lõi.
*   **Zero-downtime Migration:** Saleor áp dụng quy trình migration dữ liệu rất nghiêm ngặt. Các thay đổi về schema lớn thường được tách thành các task xử lý ngầm (background tasks) để không làm gián đoạn hệ thống đang chạy.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Thread-safe & Concurrency:** Saleor cực kỳ chú trọng vào việc tránh "race conditions".
    *   Sử dụng `F()` expressions của Django để thực hiện các phép tính toán (như tăng tồn kho) trực tiếp trên database.
    *   Sử dụng `select_for_update()` (row-level locking) để khóa bản ghi khi xử lý thanh toán hoặc đơn hàng phức tạp.
*   **Data Loaders Pattern:** Để giải quyết vấn đề N+1 đặc thù của GraphQL, Saleor sử dụng `dataloaders`. Kỹ thuật này gom các query riêng lẻ lại thành một batch query duy nhất.
*   **Metadata & Private Metadata:** Saleor cho phép mở rộng mọi model (Product, Order, User...) bằng cách thêm các trường dữ liệu động dưới dạng JSON thông qua API mà không cần sửa DB schema.
*   **Permission-driven Fields:** Sử dụng custom decorators và `PermissionsField` để kiểm soát quyền truy cập đến từng field nhỏ nhất trong schema GraphQL.

### 4. Luồng hoạt động hệ thống (System Workflow)

Ví dụ luồng xử lý khi khách hàng đặt hàng:

1.  **Request:** Client gửi một mutation GraphQL `orderCreate` qua cổng 8000 (Uvicorn).
2.  **Authentication:** Middleware kiểm tra JWT token trong Header. Nếu là App, nó kiểm tra App-Token.
3.  **Validation:** Schema GraphQL kiểm tra kiểu dữ liệu. Sau đó, logic nghiệp vụ kiểm tra tồn kho (Warehouse), giá cả (Channel) và khuyến mãi (Discount).
4.  **Transaction:** Hệ thống mở một `traced_atomic_transaction`.
    *   Tạo bản ghi Đơn hàng (Order).
    *   Khấu trừ tồn kho bằng `F()` expression để đảm bảo an toàn đa luồng.
    *   Lưu lịch sử sự kiện vào `OrderEvent`.
5.  **Event Dispatching:** `PluginsManager` (hoặc `WebhookManager`) nhận thấy sự kiện `ORDER_CREATED`.
6.  **Webhook:** Nếu có các App bên ngoài đăng ký sự kiện này, Saleor sẽ gửi một POST request (payload JSON) tới URL của App đó một cách bất đồng bộ.
7.  **Background Tasks:** Celery được kích hoạt để gửi email xác nhận cho khách hàng.
8.  **Response:** Trả về dữ liệu đơn hàng cho Client dưới dạng JSON.

### Tổng kết
Saleor là một hệ thống **Enterprise-ready**. Nó loại bỏ sự cồng kềnh của các hệ thống cũ bằng cách tập trung 100% vào API và tính thread-safe của Python. Saleor phù hợp nhất cho các thương hiệu lớn cần hiệu suất cực cao và muốn tự do tùy biến trải nghiệm người dùng cuối (Storefront).