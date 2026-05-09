Dựa trên mã nguồn và cấu trúc thư mục của dự án **Ghostfolio**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và các kỹ thuật lập trình của hệ thống quản lý tài sản mã nguồn mở này.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Stack)

Ghostfolio sử dụng mô hình **Monorepo** hiện đại, tập trung vào tính an toàn dữ liệu và khả năng mở rộng:

*   **Quản lý Workspace:** **Nx**. Đây là công cụ quan trọng giúp quản lý cả ứng dụng Backend (API) và Frontend (Client) trong cùng một kho lưu trữ, giúp chia sẻ code giữa các tầng (thư mục `libs/`) một cách dễ dàng.
*   **Backend:** **NestJS** (Node.js framework). Lựa chọn này mang lại cấu trúc Module chặt chẽ, hỗ trợ Dependency Injection mạnh mẽ và dễ dàng viết unit test.
*   **Frontend:** **Angular** (phiên bản mới nhất 21.x). Sử dụng **Angular Material** cho giao diện và **Bootstrap** cho các utility class.
*   **Cơ sở dữ liệu & ORM:** **PostgreSQL** kết hợp với **Prisma**. Prisma giúp định nghĩa schema một cách rõ ràng và tự động tạo ra các kiểu dữ liệu (Types) an toàn cho TypeScript.
*   **Caching & Background Jobs:** **Redis** và **Bull**. Hệ thống sử dụng Redis để lưu trữ cache dữ liệu thị trường và Bull để xử lý các tác vụ nặng (quét dữ liệu, tính toán danh mục) dưới nền.
*   **Xác thực:** JWT (Passport.js), hỗ trợ cả Google OAuth, OIDC và đặc biệt là **WebAuthn** (Passkeys) cho bảo mật sinh trắc học.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của Ghostfolio được thiết kế theo hướng **Domain-Driven Design (DDD)** lai với **Microservices-ready Monolith**:

*   **Tính Module hóa cực cao:** Mỗi thực thể kinh doanh (Account, Activity, Asset, Portfolio) là một module độc lập trong NestJS. Điều này cho phép tách nhỏ các logic phức tạp.
*   **Tách biệt logic tính toán (Calculator Strategy):** Trong thư mục `portfolio/calculator/`, Ghostfolio sử dụng **Factory Pattern**. Hệ thống hỗ trợ nhiều phương pháp tính toán hiệu suất danh mục khác nhau như TWR (Time-Weighted Return) và MWR (Money-Weighted Return). Người dùng có thể chuyển đổi linh hoạt mà không ảnh hưởng đến dữ liệu gốc.
*   **Lớp Common Library:** Thư mục `libs/common` chứa các DTO (Data Transfer Objects), Interface và Helper dùng chung cho cả Client và Server, đảm bảo tính nhất quán dữ liệu (Single Source of Truth).
*   **Cấu trúc Đa tầng (Layered Architecture):**
    *   *Controller:* Xử lý HTTP request/response.
    *   *Service:* Chứa logic nghiệp vụ.
    *   *Prisma Service:* Tầng truy cập dữ liệu.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Chính xác tài chính với `Big.js`:** Trong lập trình tài chính, sai số dấu phẩy động (floating point) của JavaScript là điều tối kỵ. Ghostfolio sử dụng thư viện `big.js` cho mọi phép tính số dư và tỷ giá để đảm bảo độ chính xác tuyệt đối.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Sử dụng `EventEmitter2`. Khi một hoạt động (Activity) được tạo, hệ thống phát đi sự kiện `PortfolioChangedEvent`. Các listener sẽ tự động cập nhật lại snapshot danh mục hoặc xóa cache liên quan mà không làm chậm request chính.
*   **Hệ thống Interceptor thông minh:**
    *   `RedactValuesInResponseInterceptor`: Tự động ẩn các giá trị nhạy cảm trong response nếu người dùng đang ở chế độ demo hoặc chia sẻ công khai.
    *   `LogPerformance`: Decorator tùy chỉnh để đo lường thời gian thực thi của các hàm tính toán tài chính phức tạp.
*   **Hệ thống phân quyền dựa trên Permission:** Thay vì chỉ kiểm tra Role (Admin/User), dự án sử dụng `HasPermissionGuard` để kiểm tra quyền truy cập chi tiết (ví dụ: `createAccess`, `updateActivity`), giúp việc mở rộng các gói tính năng (Premium/Basic) trở nên linh hoạt.
*   **Data Gathering Pipeline:** Quy trình thu thập dữ liệu thị trường được thiết kế để chịu lỗi (resilient). Nếu một nhà cung cấp (Yahoo Finance, CoinGecko) gặp sự cố, hệ thống sẽ sử dụng dữ liệu từ cache hoặc chuyển sang provider dự phòng.

---

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Thu thập dữ liệu thị trường:**
    *   `CronService` kích hoạt các job định kỳ.
    *   `DataGatheringProcessor` gọi các API tài chính bên ngoài.
    *   Dữ liệu được lưu vào bảng `MarketData` và cập nhật vào `Redis`.
2.  **Xử lý giao dịch người dùng:**
    *   User nhập lệnh BUY/SELL qua API hoặc giao diện.
    *   `ActivitiesService` kiểm tra tính hợp lệ và lưu vào DB qua Prisma.
    *   Một sự kiện "Portfolio Changed" được bắn ra.
3.  **Tính toán danh mục (On-demand hoặc Background):**
    *   Khi User truy cập dashboard, `PortfolioService` sẽ gọi `PortfolioCalculator`.
    *   Calculator lấy dữ liệu lịch sử giá từ bảng `MarketData` và các giao dịch từ bảng `Order`.
    *   Áp dụng công thức tài chính để trả về ROAI (Return on Average Investment).
4.  **Hiển thị Real-time:**
    *   Angular Client nhận dữ liệu qua REST API.
    *   Sử dụng RxJS để quản lý luồng dữ liệu và cập nhật các biểu đồ (Chart.js) một cách mượt mà.

---

### 5. Tổng kết

Ghostfolio là một dự án **Production-grade** thực thụ. Nó không chỉ là một ứng dụng quản lý tài chính đơn thuần mà còn là một ví dụ mẫu mực về cách xây dựng hệ thống **Fullstack TypeScript** an toàn, chính xác và dễ bảo trì. Việc kết hợp giữa Nx, NestJS và Prisma tạo ra một "tam giác vàng" giúp tối ưu hóa năng suất phát triển và chất lượng mã nguồn.