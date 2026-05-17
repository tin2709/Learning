Dựa trên tài liệu và cấu trúc mã nguồn của repository **HyperDX**, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và các kỹ thuật đặc sắc của dự án này.

---

### 1. Công nghệ Cốt lõi (Core Technology Stack)

HyperDX là một nền tảng quan sát (observability) hiện đại, tận dụng tối đa hệ sinh thái TypeScript:

*   **Ngôn ngữ & Runtime**: TypeScript chủ đạo (94.5%), chạy trên Node.js 22+.
*   **Lưu trữ (Dữ liệu kép)**:
    *   **ClickHouse**: Đóng vai trò là "trái tim" để lưu trữ và truy vấn dữ liệu telemetry (logs, metrics, traces) với hiệu suất cực cao.
    *   **MongoDB (Mongoose)**: Lưu trữ metadata như thông tin người dùng, cấu hình dashboard, định nghĩa alert và saved search.
*   **Frontend**: Next.js 14, Mantine UI (component library), TanStack Query (quản lý server state), và Jotai (quản lý client state).
*   **Ingestion (Tiếp nhận dữ liệu)**: OpenTelemetry (OTel) Collector (viết bằng Go và cấu hình YAML) để thu thập dữ liệu từ các ứng dụng.
*   **Công cụ Monorepo**: Quản lý bằng **Nx** và **Yarn 4 (Berry)**, giúp tối ưu hóa việc build và quản lý dependency giữa `packages/api`, `packages/app` và `packages/common-utils`.

---

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của HyperDX tập trung vào việc **thống nhất (unification)** và **hiệu suất (performance)**:

*   **Unified Observability**: Thay vì tách rời logs, metrics và traces, HyperDX cho phép tìm kiếm và tương quan (correlate) tất cả chúng tại một nơi duy nhất.
*   **Schema Agnostic (Không phụ thuộc lược đồ)**: Hệ thống được thiết kế để hoạt động dựa trên bất kỳ cấu trúc dữ liệu nào trong ClickHouse, thay vì bắt buộc người dùng tuân theo một schema cứng nhắc.
*   **Multi-tenancy (Đa người dùng)**: Toàn bộ dữ liệu được phân quyền theo `Team`. Mọi truy vấn từ backend vào database đều được bọc bởi lớp filter theo team ID để đảm bảo cô lập dữ liệu.
*   **AI-Native & MCP (Model Context Protocol)**: HyperDX tích hợp sẵn MCP Server, cho phép các AI Agent (như Claude Code, Cursor) trực tiếp truy vấn dữ liệu quan sát và quản lý dashboard thông qua các "công cụ" (tools) tiêu chuẩn hóa.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Notable Programming Techniques)

*   **Slot-based Port Isolation (Kỹ thuật cô lập cổng)**: Trong `Makefile` và `scripts/dev-env.sh`, HyperDX sử dụng hàm `cksum` dựa trên tên thư mục làm việc để tính toán ra một "slot" (0-99). Từ slot này, hệ thống tự động gán các cổng (port) duy nhất cho các service (API, App, DB). Điều này cho phép nhiều lập trình viên hoặc AI Agent chạy nhiều môi trường dev/test trên cùng một máy chủ mà không bao giờ bị xung đột cổng.
*   **Safe SQL Building**: Sử dụng package `common-utils` để xây dựng các câu lệnh SQL ClickHouse một cách an toàn, tránh lỗi SQL Injection và tối ưu hóa việc sử dụng Materialized Views để tăng tốc độ truy vấn.
*   **Inlined API cho Vercel Previews**: Kỹ thuật đóng gói toàn bộ Express API vào trong Next.js serverless function khi triển khai trên Vercel Preview. Điều này giúp các bản xem trước có đầy đủ tính năng backend mà không cần triển khai một cụm API riêng biệt.
*   **Complex Event Parsing**: Sử dụng **ANTLR** để định nghĩa bộ ngữ pháp tìm kiếm (`SearchGrammar.g4`), cho phép người dùng sử dụng cú pháp tìm kiếm tự nhiên (như `level:err`) thay vì phải viết SQL thuần.

---

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng Thu thập Dữ liệu (Ingestion Path):
1.  **Ứng dụng khách**: Gửi dữ liệu qua OTLP (gRPC/HTTP).
2.  **OTel Collector**: Tiếp nhận, xử lý (parse JSON, lọc dữ liệu) thông qua các processor.
3.  **ClickHouse**: Lưu trữ dữ liệu thô và cập nhật các Materialized Views để phục vụ rollup dữ liệu theo thời gian (15 phút, 1 giờ...).

#### B. Luồng Truy vấn & Cảnh báo (Query & Alerting):
1.  **UI**: Gửi yêu cầu truy vấn thông qua TanStack Query tới API.
2.  **API**: Kiểm tra quyền team, sau đó sử dụng bộ parser để chuyển đổi search query thành SQL.
3.  **ClickHouse Proxy**: API thực thi SQL trên ClickHouse và trả kết quả về frontend.
4.  **Alert Task**: Một worker chạy ngầm định kỳ quét ClickHouse. Nếu dữ liệu vượt ngưỡng (threshold), nó sẽ kích hoạt webhook hoặc thông báo qua Slack/Discord.

#### C. Luồng AI-Assisted (Tương tác với AI):
1.  **AI Agent**: Gọi công cụ qua giao thức MCP.
2.  **HyperDX MCP Server**: Thực thi logic (ví dụ: `hyperdx_query`) và trả về ngữ cảnh (context) cho AI để nó có thể giải thích lỗi hoặc đề xuất cách sửa code.

---

**Tổng kết**: HyperDX là một hệ thống tinh vi kết hợp giữa sức mạnh xử lý dữ liệu lớn của ClickHouse và sự linh hoạt của hệ sinh thái JavaScript/AI, với những kỹ thuật quản lý hạ tầng dev (port isolation) cực kỳ sáng tạo.