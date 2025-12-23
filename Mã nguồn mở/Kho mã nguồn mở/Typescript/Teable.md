Dựa trên cấu trúc thư mục và mã nguồn của dự án **Teable**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và quy trình kỹ thuật của nền tảng này bằng tiếng Việt.

---

### 1. Tổng quan: Teable là gì?
Teable không chỉ là một bảng tính (spreadsheet) đơn thuần; nó tự định nghĩa là **"No-code Postgres"**. Mục tiêu của dự án là cung cấp giao diện thân thiện như Airtable nhưng chạy trên nền tảng cơ sở dữ liệu SQL chuyên nghiệp, cho phép mở rộng lên đến hàng triệu dòng dữ liệu mà không làm giảm hiệu năng.

### 2. Công nghệ cốt lõi (Tech Stack)
Dự án được xây dựng theo mô hình **Monorepo** sử dụng **pnpm workspaces**:

*   **Backend:** Chạy trên framework **NestJS** (Node.js). Đây là kiến trúc hướng module, rất mạnh mẽ cho các hệ thống doanh nghiệp.
*   **Frontend:** Sử dụng **Next.js** (React) cùng với **Tailwind CSS** và thư viện UI **Shadcn UI**.
*   **Cơ sở dữ liệu (Quản lý 2 tầng):**
    *   **Prisma:** Dùng để quản lý dữ liệu hệ thống (Metadata) như: thông tin người dùng, quyền truy cập, cấu trúc các Workspace.
    *   **Knex.js:** Dùng để thực thi các truy vấn động lên dữ liệu của người dùng. Khi bạn tạo một "Table" trên giao diện, Teable dùng Knex để tạo một bảng vật lý thật sự trong Postgres.
*   **Xử lý thời gian thực (Real-time):** Sử dụng **ShareDB** với thuật toán **OT (Operational Transformation)**. Điều này cho phép nhiều người cùng sửa một ô dữ liệu mà không bị xung đột (giống Google Docs).
*   **Engine công thức:** Sử dụng **ANTLR4** (`Formula.g4`) để phân tích các công thức kiểu Excel và biên dịch chúng trực tiếp thành câu lệnh SQL thuần túy.

---

### 3. Tư duy kiến trúc & Kỹ thuật chính

#### A. Kiến trúc CSDL tầng kép (Dual-Layer)
Teable giải quyết bài toán hiệu năng bằng cách tách biệt:
1.  **Lớp Meta (Blueprint):** Lưu trữ định nghĩa bảng, kiểu dữ liệu của cột, quyền hạn (do Prisma quản lý).
2.  **Lớp Data (Physical Reality):** Mỗi bảng của người dùng là một bảng SQL thực sự. Điều này cho phép tận dụng tối đa Index và khả năng tối ưu của Postgres thay vì lưu mọi thứ vào một bảng JSON khổng lồ như các công cụ no-code khác.

#### B. Áp dụng Visitor Pattern trong SQL Generation
Trong mã nguồn (ví dụ: `create-database-column-field-visitor.postgres.ts`), Teable áp dụng **Design Pattern: Visitor**.
*   **Tại sao?** Vì mỗi loại trường (Link, Công thức, Rollup, Attachment) có cách tạo cột SQL khác nhau.
*   **Hoạt động:** Hệ thống sẽ "duyệt" qua các loại trường và sinh ra mã DDL (`ALTER TABLE ADD COLUMN...`) tương ứng cho từng loại DB (Postgres hoặc SQLite).

#### C. Biên dịch công thức xuống Database
Thay vì tính toán công thức bằng JavaScript (chậm), Teable biên dịch công thức của người dùng thành **Generated Columns** (Stored) trong Postgres.
*   Ví dụ: Nếu bạn có công thức `{Giá} * {Số lượng}`, Teable sẽ tạo một cột trong Postgres là `GENERATED ALWAYS AS (price * quantity) STORED`.
*   **Lợi ích:** Tốc độ tính toán đạt mức tối đa của Database, không tốn tài nguyên server Node.js.

#### D. Quản lý tính toàn vẹn (Integrity)
Dự án có các module riêng trong `src/db-provider/integrity-query/` để xử lý ràng buộc dữ liệu. Khi bạn tạo liên kết (Link) giữa hai bảng, Teable tự động quản lý các Foreign Keys và Junction Tables (bảng trung gian cho quan hệ N-N) một cách tự động ở tầng DB.

---

### 4. Tóm tắt luồng hoạt động (Workflow)

1.  **Người dùng thao tác:** Ví dụ: "Thêm một dòng mới".
2.  **Validation (Zod):** Request đi qua `zod.validation.pipe.ts` để đảm bảo dữ liệu đúng kiểu (ví dụ: cột số không được nhập chữ).
3.  **Xử lý OT (Real-time):** ShareDB ghi nhận thay đổi và phát tán (broadcast) đến các người dùng khác đang xem bảng đó qua WebSocket.
4.  **DbProvider thực thi:** `postgres.provider.ts` chuyển đổi hành động đó thành câu lệnh SQL (INSERT).
5.  **Tính toán lại (Cascade Update):** Nếu dòng vừa thêm có ảnh hưởng đến các cột công thức hoặc bảng khác đang liên kết, `CalculationModule` sẽ kích hoạt luồng cập nhật dây chuyền để đảm bảo dữ liệu toàn hệ thống luôn khớp nhau.

---

### 5. Điểm mạnh kỹ thuật nổi bật
*   **Khả năng mở rộng (Extensibility):** Thư mục `packages/sdk` và `plugins/` cho thấy Teable được thiết kế như một nền tảng (Platform) để bên thứ ba có thể viết thêm tính năng.
*   **Giám sát chuyên nghiệp (Observability):** Tích hợp sẵn **OpenTelemetry** và **Sentry** (`tracing.ts`), cho thấy dự án hướng tới môi trường chạy thực tế (production) quy mô lớn.
*   **Trung lập về Database:** Cấu trúc `DbProvider` cho phép trong tương lai có thể hỗ trợ thêm MySQL, SQL Server hoặc ClickHouse chỉ bằng cách thêm một file Provider mới.
