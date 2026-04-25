Dựa trên nội dung mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Flexprice** - một hệ thống hạ tầng thanh toán (monetization infrastructure) hiện đại.

---

### 1. Công nghệ cốt lõi (Core Technology)

Flexprice sử dụng một "stack" công nghệ mạnh mẽ, được thiết kế để xử lý dữ liệu lớn (big data) và đảm bảo tính nhất quán của giao dịch tài chính:

*   **Ngôn ngữ lập trình:** **Go (Golang) 1.23+**. Lựa chọn này tối ưu cho hiệu năng cao, xử lý đồng thời (concurrency) tốt và tiết kiệm tài nguyên hệ thống.
*   **Cơ sở dữ liệu (Database Strategy):**
    *   **PostgreSQL:** Dùng cho các giao dịch nghiệp vụ (transactional data) như thông tin khách hàng, gói cước (plans), đăng ký (subscriptions) và hóa đơn (invoices). Sử dụng **Ent ORM** để quản lý schema và query.
    *   **ClickHouse:** Dùng cho dữ liệu sự kiện (usage events) và phân tích (analytics). ClickHouse là OLAP database cực nhanh, phù hợp để lưu trữ hàng triệu "metering events" và thực hiện các phép tính tổng hợp (aggregation) theo thời gian thực.
*   **Hệ thống tin nhắn (Messaging):** **Kafka**. Đây là "xương sống" để thu thập dữ liệu sử dụng từ khách hàng một cách bất đồng bộ, đảm bảo hệ thống API không bị nghẽn khi có lượng lớn sự kiện đổ về.
*   **Điều phối quy trình (Workflow Orchestration):** **Temporal**. Đây là điểm đặc biệt nhất. Temporal quản lý các quy trình kéo dài (long-running) như chu kỳ thanh toán hàng tháng, gia hạn gói cước, hoặc xử lý nợ, đảm bảo quy trình luôn hoàn thành ngay cả khi có lỗi hệ thống xảy ra giữa chừng.
*   **Dependency Injection:** **Uber FX**. Giúp quản lý các thành phần (services, repositories) một cách module hóa và dễ kiểm thử.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Flexprice tuân thủ các nguyên lý thiết kế hệ thống phân tán hiện đại:

*   **Kiến trúc phân lớp (Layered Architecture):**
    *   **Domain Layer:** Chứa business logic thuần túy và định nghĩa interface. Không phụ thuộc vào database cụ thể.
    *   **Repository Layer:** Triển khai việc truy xuất dữ liệu (Postgres via Ent hoặc ClickHouse).
    *   **Service Layer:** Điều phối logic nghiệp vụ, quản lý transaction và tương tác với các hệ thống bên ngoài.
    *   **API/Controller Layer:** Tiếp nhận request, validate DTOs và trả về response.
*   **Tách biệt đọc/ghi sự kiện (Event Sourcing influence):** Thay vì cập nhật trực tiếp số dư khi có sự kiện, hệ thống lưu mọi sự kiện vào Kafka/ClickHouse và tính toán "Usage" dựa trên các sự kiện đó khi đến kỳ thanh toán.
*   **Multi-Tenancy & Environments:** Mọi dữ liệu đều được cô lập theo `TenantID` và `EnvironmentID` (Sandbox vs Production), cho phép một instance phục vụ nhiều khách hàng (B2B) với nhiều môi trường thử nghiệm khác nhau.
*   **Mô hình triển khai linh hoạt (Split Deployment):** Có thể chạy toàn bộ trong 1 process (`local` mode) hoặc tách thành các service riêng biệt: `flexprice-api`, `flexprice-consumer` (xử lý Kafka), và `flexprice-worker` (xử lý Temporal).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Real-time Metering (Đo lường thời gian thực):** Kỹ thuật tính toán Usage dựa trên các "Meters" (định nghĩa cách tính: COUNT, SUM, MAX...). Hệ thống query trực tiếp vào ClickHouse để lấy dữ liệu tổng hợp tại thời điểm lập hóa đơn.
*   **Idempotency (Tính bất biến):** Sử dụng `Idempotency-Key` trong API để đảm bảo nếu khách hàng gửi yêu cầu tạo hóa đơn hoặc thanh toán nhiều lần do lỗi mạng, hệ thống cũng chỉ xử lý duy nhất một lần.
*   **Credit/Wallet Management:** Kỹ thuật quản lý ví điện tử (Wallets) với các khoản Credit Grant (tặng tiền) có thời hạn hết hạn. Logic trừ tiền được ưu tiên theo thứ tự: Credit (khuyến mãi) -> Prepaid (trả trước) -> Postpaid (trả sau).
*   **Automated SDK Generation:** Sử dụng **Speakeasy** để tự động tạo SDK cho Go, Python, TypeScript từ file OpenAPI. Điều này giúp đảm bảo sự đồng nhất giữa Backend và các thư viện khách hàng sử dụng.
*   **Draft-First Invoicing:** Một kỹ thuật an toàn trong kế toán: Luôn tạo hóa đơn nháp (Draft) -> Tính toán (Compute) -> Kiểm tra (Validate) -> Sau đó mới chốt (Finalize) để phát hành số hóa đơn chính thức.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Hệ thống hoạt động theo 3 luồng chính:

#### A. Luồng đẩy dữ liệu sử dụng (Ingestion Flow):
1.  **App của người dùng** gửi `Event` (ví dụ: "api_call") qua SDK/API.
2.  **API Server** nhận sự kiện, đẩy vào **Kafka**.
3.  **Kafka Consumer** đọc sự kiện, chuẩn hóa dữ liệu và lưu vào **ClickHouse**.

#### B. Luồng tính cước và tạo hóa đơn (Billing Flow):
1.  **Temporal Cron** kích hoạt khi đến chu kỳ thanh toán của một `Subscription`.
2.  **Worker** truy vấn **ClickHouse** để lấy tổng lượng sử dụng (Usage) dựa trên các `Meters`.
3.  Áp dụng **Plan & Price** (bậc thang, cố định, hoặc theo đơn vị) để tính tiền.
4.  Áp dụng **Coupons/Discounts** và trừ tiền từ **Wallet**.
5.  Tạo **Invoice** nháp, sau đó finalize và gửi Webhook/Email cho khách hàng.

#### C. Luồng quản lý ví và hạn mức (Entitlement Flow):
1.  Khi người dùng thực hiện một hành động, App gọi API `check_entitlement`.
2.  Flexprice kiểm tra gói cước hiện tại và lượng usage đã dùng trong ClickHouse.
3.  Trả về kết quả cho phép hoặc từ chối hành động (Feature Gating).

### Kết luận
Flexprice là một hệ thống **Middleware** phức tạp, giải quyết bài toán khó nhất của SaaS: **"Làm sao để tính tiền chính xác dựa trên việc sử dụng thực tế mà vẫn đảm bảo hiệu năng?"**. Việc kết hợp giữa ClickHouse (tốc độ xử lý dữ liệu) và Temporal (độ tin cậy quy trình) làm cho kiến trúc này cực kỳ đáng tin cậy cho các startup AI/SaaS quy mô lớn.