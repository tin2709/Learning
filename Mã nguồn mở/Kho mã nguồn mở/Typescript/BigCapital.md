Dựa trên cấu trúc thư mục và nội dung các tệp tin của dự án **Bigcapital**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng kế toán mã nguồn mở này:

### 1. Công nghệ cốt lõi (Core Technology)

Dự án được xây dựng theo mô hình **Monorepo** hiện đại, sử dụng các công cụ mạnh mẽ nhất trong hệ sinh thái JavaScript/TypeScript:

*   **Backend (Server):** Sử dụng framework **NestJS** (Node.js). Đây là một framework hướng đối tượng, giúp mã nguồn có tính cấu trúc cao.
*   **Database & ORM:** 
    *   Sử dụng **MariaDB** làm cơ sở dữ liệu chính.
    *   Sử dụng **Objection.js** kết hợp với **Knex.js**. Đây là lựa chọn linh hoạt cho các truy vấn SQL phức tạp thường gặp trong kế toán, thay vì dùng các ORM quá trừu tượng như TypeORM hay Prisma.
*   **Frontend (Webapp):** Sử dụng **React** kết hợp với **Vite** (giúp tốc độ build nhanh hơn nhiều so với CRA truyền thống). Quản lý state bằng **Redux**.
*   **Proxy & Network:** Sử dụng **Envoy Proxy** thay thế cho Nginx để điều phối traffic giữa frontend và backend trong môi trường Docker.
*   **Hạ tầng:** 
    *   **Docker & Docker Compose** là phương thức triển khai chủ đạo.
    *   **Redis** được dùng để quản lý hàng đợi (Queue) thông qua **BullMQ**.
    *   **Gotenberg** được dùng làm engine để render PDF (hóa đơn, báo cáo).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Bigcapital tập trung vào khả năng mở rộng cho doanh nghiệp (Enterprise-ready):

*   **Kiến trúc Đa thuê (Multi-tenancy):** Hệ thống phân tách rõ rệt giữa cơ sở dữ liệu `System` (quản lý người dùng, tenant, subscription) và cơ sở dữ liệu `Tenant` (dữ liệu kế toán riêng biệt cho từng công ty). Mỗi tenant thường có tiền tố database riêng (ví dụ: `bigcapital_tenant_1`).
*   **Repository Pattern:** Sử dụng các class như `EntityRepository`, `TenantRepository` để trừu tượng hóa lớp dữ liệu, giúp logic nghiệp vụ không bị phụ thuộc cứng vào cấu trúc bảng.
*   **Event-Driven (Kiến trúc hướng sự kiện):** NestJS EventEmitter được sử dụng rộng rãi. Khi một giao dịch (Invoice, Bill) được tạo, hệ thống sẽ phát ra sự kiện để các **Subscribers** thực hiện các tác vụ phụ (như ghi sổ cái, cập nhật kho) mà không làm chậm luồng chính.
*   **Headless Accounting:** Backend được thiết kế như một API Engine độc lập. SDK TypeScript (`shared/sdk-ts`) cho phép các hệ thống khác tích hợp sâu vào hệ thống kế toán kép của Bigcapital.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Hệ thống kế toán kép (Double-Entry System):** Mọi giao dịch tài chính cuối cùng đều được chuyển đổi thành các bút toán Nợ (Debit) và Có (Credit) trong bảng `accounts_transactions`.
*   **Xử lý tiền tệ & Tỷ giá:** Tích hợp **Open Exchange Rates** để cập nhật tỷ giá tự động và hỗ trợ đa tiền tệ (Multi-currency) sâu trong sổ cái.
*   **Quản lý kho theo lô (Inventory Costing):** Sử dụng phương pháp **Average Cost Method** (Giá bình quân) và theo dõi thông qua `InventoryCostLotTracker`.
*   **Interceptors & Pipes:** Sử dụng kỹ thuật chuyển đổi `camelCase` (frontend) sang `snake_case` (database) tự động thông qua `SerializeInterceptor`, giúp code sạch và tuân thủ tiêu chuẩn của từng lớp.
*   **Báo cáo tài chính động:** Các báo cáo như Bảng cân đối kế toán (Balance Sheet) hay P&L được tính toán dựa trên các công thức định nghĩa trong `FinancialStatements/common/FinancialEvaluateEquation.ts`.

### 4. Tóm tắt luồng hoạt động (Activity Flow Summary)

Luồng xử lý một giao dịch điển hình (ví dụ: Tạo hóa đơn bán hàng):

1.  **Request:** Người dùng thao tác trên giao diện React, gửi dữ liệu về NestJS API.
2.  **Validation:** `ZodValidation.pipe.ts` hoặc `ClassValidation.pipe.ts` kiểm tra tính hợp lệ của dữ liệu đầu vào.
3.  **Service Logic:** `SaleInvoices.service.ts` xử lý lưu trữ hóa đơn vào database của Tenant tương ứng.
4.  **Event Trigger:** Một sự kiện `onSaleInvoiceCreated` được phát ra.
5.  **Side Effects (Subscribers):**
    *   **G/L Writing:** `InvoiceGLEntriesSubscriber` sẽ tự động tạo các bút toán nợ/có tương ứng vào sổ cái.
    *   **Inventory Update:** Nếu hóa đơn có hàng hóa, một subscriber khác sẽ cập nhật tồn kho và tính toán lại giá vốn hàng bán.
6.  **Notification:** Nếu được cấu hình, hệ thống sẽ gửi email (qua `MailNotification`) hoặc SMS cho khách hàng.
7.  **Reporting:** Khi người dùng xem báo cáo, `FinancialReportService` sẽ truy vấn toàn bộ bảng `accounts_transactions`, nhóm theo tài khoản và thời gian để trả về dữ liệu tài chính thời gian thực.