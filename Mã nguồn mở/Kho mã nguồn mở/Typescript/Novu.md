Dựa trên cấu trúc thư mục và mã nguồn của kho lưu trữ **Novu** mà bạn cung cấp, dưới đây là bản phân tích chi tiết bằng tiếng Việt về công nghệ, kiến trúc và luồng hoạt động của dự án này.

---

# Phân tích Hệ thống hạ tầng thông báo Novu

Novu là một nền tảng hạ tầng thông báo (Notification Infrastructure) mã nguồn mở, giúp tập trung hóa việc gửi thông báo qua nhiều kênh (Email, SMS, Push, In-app, Chat) vào một API duy nhất.

## 1. Công nghệ cốt lõi (Core Technology Stack)

Novu sử dụng mô hình **Monorepo** quản lý bởi **Nx** và **pnpm**, cho phép quản lý hàng chục package và ứng dụng trong một nơi.

*   **Backend:**
    *   **Ngôn ngữ:** TypeScript (chiếm >96%).
    *   **Framework:** **NestJS** (Dựa trên Node.js) - Sử dụng tính chất Modular để chia các service (API, Worker, Webhook, WS).
    *   **Cơ sở dữ liệu:** **MongoDB** (Lưu trữ dữ liệu chính như User, Template, Organization) và **Redis** (Làm hàng đợi và Cache).
    *   **Xử lý hàng đợi:** **BullMQ** - Dùng để quản lý các tác vụ gửi thông báo nặng, đảm bảo tính bất đồng bộ và khả năng retry.
    *   **Analytics:** **ClickHouse** - Lưu trữ log và dữ liệu phân tích với quy mô lớn.
*   **Frontend:**
    *   **React 18** kết hợp với **Vite** (cho Dashboard mới) và CRA (cho Dashboard cũ).
    *   **Styling:** **Tailwind CSS**, **Panda CSS** và **Radix UI** (Headless UI).
    *   **State Management:** **TanStack Query (React Query)** để quản lý dữ liệu từ API.
*   **DevOps & Tools:**
    *   **Docker & Docker Compose:** Cung cấp môi trường chạy local nhanh chóng.
    *   **Biome:** Dùng để Linting và Formatting code (thay thế ESLint/Prettier để đạt tốc độ cao).
    *   **Speakeasy:** Tự động tạo SDK từ cấu hình OpenAPI.

## 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Novu được thiết kế theo hướng **Scalable (Mở rộng)** và **Provider-agnostic (Không phụ thuộc nhà cung cấp)**:

*   **Kiến trúc Usecase-Driven (CQRS-lite):** Thay vì viết logic trực tiếp ở Controller, Novu tách biệt các luồng xử lý vào các `usecases` và `commands`. Mỗi hành động (gửi mail, tạo template) là một Usecase riêng biệt, cực kỳ dễ kiểm thử (Unit Test) và tái sử dụng.
*   **Abstraction Layer (Lớp trừu tượng):** Novu tạo ra một lớp trung gian giữa ứng dụng và các nhà cung cấp thông báo. Bạn chỉ cần gọi API của Novu, còn việc chuyển đổi thành định dạng của SendGrid, Twilio hay Firebase sẽ do lớp `packages/providers` đảm nhiệm.
*   **Open Core:** Phân tách rõ ràng giữa mã nguồn mở (MIT) và các tính năng doanh nghiệp (`enterprise` folder) như RBAC, Single Sign-On, Billing.
*   **Environment & Organization Isolation:** Dữ liệu luôn được phân tách nghiêm ngặt theo `organizationId` và `environmentId` (Development vs Production).

## 3. Các kỹ thuật chính (Key Techniques)

*   **Workflow Engine:** Đây là "trái tim" của hệ thống. Cho phép định nghĩa các luồng thông báo phức tạp (ví dụ: Chờ 2 giờ -> Nếu chưa đọc In-app -> Gửi Email).
*   **Digest Engine:** Kỹ thuật gom nhiều thông báo nhỏ thành một thông báo tổng hợp (ví dụ: "Bạn có 5 tin nhắn mới" thay vì gửi 5 email riêng lẻ).
*   **Stateless Provider:** Cho phép gửi thông báo mà không cần lưu trữ trạng thái phức tạp, tối ưu hóa hiệu suất xử lý luồng (trong `packages/stateless`).
*   **In-app Realtime:** Sử dụng **WebSockets (Socket.io)** để đẩy thông báo trực tiếp lên giao diện người dùng ngay lập tức.
*   **Template Parsing:** Sử dụng **Handlebars** và **LiquidJS** để cá nhân hóa nội dung thông báo dựa trên dữ liệu động (payload).
*   **Idempotency:** Kỹ thuật đảm bảo một thông báo không bị gửi lặp lại nhiều lần nếu API bị gọi trùng (thông qua `idempotency-key`).

## 4. Tóm tắt luồng hoạt động (Activity Flow)

Luồng đi của một thông báo trong Novu diễn ra như sau:

1.  **Trigger (Kích hoạt):** Hệ thống phía Client (Backend của bạn) gọi API `v1/events/trigger` với một `workflowId` và `payload` dữ liệu.
2.  **Ingestion (Tiếp nhận):** **API Service** tiếp nhận yêu cầu, kiểm tra xác thực, định danh Subscriber (người nhận) và lưu vào database.
3.  **Queuing (Xếp hàng):** API đẩy một Job vào hàng đợi **Redis (BullMQ)**.
4.  **Processing (Xử lý - Worker):** **Worker Service** lấy Job ra, phân tích Workflow:
    *   Kiểm tra tùy chọn của người dùng (họ có tắt thông báo kênh này không?).
    *   Tính toán thời gian trễ (Delay) hoặc gom nhóm (Digest).
    *   Render nội dung từ Template bằng Payload dữ liệu.
5.  **Delivery (Gửi đi):** Worker gọi đến **Provider Manager**. Tại đây, hệ thống chọn nhà cung cấp phù hợp (ví dụ: Mailgun) và thực hiện gửi tin thực tế.
6.  **Tracking & Analytics (Theo dõi):** Sau khi gửi, trạng thái (Gửi thành công, Thất bại, Đã mở) được ghi lại vào **MongoDB** và **ClickHouse** để hiển thị trên Dashboard.
7.  **Realtime Update:** Nếu có kênh In-app, **WS Service** sẽ gửi tín hiệu qua WebSocket để biểu tượng thông báo trên Web/App của người dùng sáng lên.

---
**Kết luận:** Novu là một dự án có kiến trúc rất bài bản, áp dụng các chuẩn mực hiện đại của Node.js/TypeScript. Việc chia nhỏ các Service (API, Worker, WS) giúp hệ thống có thể chịu tải cực cao bằng cách scale riêng lẻ từng bộ phận.