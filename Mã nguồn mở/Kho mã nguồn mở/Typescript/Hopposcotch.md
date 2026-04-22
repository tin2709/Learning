Hoppscotch là một hệ sinh thái phát triển API mã nguồn mở (thay thế cho Postman) với kiến trúc hiện đại, tập trung vào hiệu suất và tính linh động trên nhiều nền tảng. Dưới đây là phân tích chi tiết dựa trên mã nguồn của dự án:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ & Framework chính:**
    *   **Frontend:** Vue 3 với TypeScript, sử dụng Vite làm công cụ build. Dự án tận dụng mạnh mẽ Composition API để quản lý logic.
    *   **Backend:** NestJS (Node.js framework) – một kiến trúc hướng module, cung cấp cả REST API và GraphQL (Apollo Server).
    *   **Desktop App:** Tauri (Rust + Webview) – thay vì Electron cồng kềnh, Tauri giúp ứng dụng desktop cực kỳ nhẹ và bảo mật.
    *   **CLI:** TypeScript, dùng để chạy các bộ test API trong môi trường CI/CD.
*   **Quản lý dữ liệu:**
    *   **ORM:** Prisma – giúp tương tác với cơ sở dữ liệu PostgreSQL một cách type-safe.
    *   **Cơ sở dữ liệu:** PostgreSQL là lựa chọn chính cho bản self-host.
*   **Giao thức hỗ trợ:** REST, GraphQL, WebSocket, Server-Sent Events (SSE), Socket.IO, và MQTT.
*   **Hạ tầng:** Docker, Caddy (Web server & Reverse Proxy), Firebase (cho bản Cloud).

### 2. Tư duy kiến trúc (Architectural Thinking)

Dự án sử dụng mô hình **Monorepo** (quản lý bởi `pnpm`), chia hệ thống thành các gói (`packages/`) độc lập nhưng có liên kết chặt chẽ:

*   **Kiến trúc Phẳng & Module:** Hoppscotch chia nhỏ logic thành các package như `hoppscotch-common` (chứa UI và logic lõi dùng chung), `hoppscotch-backend`, `hoppscotch-agent`, và `hoppscotch-cli`.
*   **Platform Independent (Độc lập nền tảng):** Logic xử lý request được tách rời khỏi lớp hiển thị. Điều này cho phép Hoppscotch chạy mượt mà trên Trình duyệt (PWA), Desktop (Tauri) và Terminal (CLI).
*   **Kiến trúc All-in-One (AIO):** Qua file `prod.Dockerfile` và `aio_run.mjs`, dự án cung cấp khả năng đóng gói toàn bộ hệ thống (Frontend, Backend, Admin) vào một container duy nhất, giúp việc tự triển khai (self-hosting) cực kỳ đơn giản.
*   **Cơ chế Interceptor (Người trung gian):** Để vượt qua giới hạn CORS của trình duyệt, dự án thiết kế các "Interceptors" như Proxy, Browser Extension, hoặc Hoppscotch Agent (Rust).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Lập trình hàm (Functional Programming):** Sử dụng thư viện `fp-ts` rất nhiều (Either, Option, TaskEither, pipe). Kỹ thuật này giúp quản lý lỗi và luồng dữ liệu một cách chặt chẽ, tránh lỗi runtime không đáng có.
*   **Type-Safe tuyệt đối:** Tận dụng tối đa TypeScript từ Frontend đến Backend. Prisma tự động sinh ra các kiểu dữ liệu cho DB, trong khi GraphQL Code Generator giúp đồng bộ kiểu giữa Client và Server.
*   **Reactivity linh hoạt:** Ngoài hệ thống reactivity của Vue, dự án còn tự xây dựng các store riêng (trong `newstore/`) để quản lý trạng thái phức tạp như lịch sử request và cấu hình môi trường.
*   **Bảo mật dữ liệu nhạy cảm:** Sử dụng mã hóa AES-256-GCM (thấy trong package Rust/Tauri) để bảo mật các khóa API và thông tin xác thực lưu trữ cục bộ.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Mô tả luồng khi người dùng gửi một request API:

1.  **Giai đoạn Chuẩn bị (Frontend):** Người dùng nhập URL, Header, Body. Vue frontend sử dụng các helper trong `hoppscotch-common` để xây dựng object request.
2.  **Giai đoạn Chặn/Chuyển hướng (Interceptor):** 
    *   Nếu dùng trình duyệt thuần, request sẽ bị giới hạn bởi CORS.
    *   Nếu dùng **Hoppscotch Agent**, request sẽ được gửi tới một server local chạy bằng Rust (Tauri) qua port 9119. Agent này thực hiện request thật sự từ hệ điều hành để vượt qua mọi rào cản trình duyệt.
3.  **Giai đoạn Xử lý Script:** 
    *   **Pre-request Script:** Chạy code JS trước khi gửi (ví dụ: tạo token).
    *   **Execution:** Gửi request.
    *   **Post-request (Tests):** Nhận response và chạy các đoạn mã kiểm thử để xác nhận trạng thái (Status code, body content).
4.  **Giai đoạn Lưu trữ & Đồng bộ:** 
    *   Request được lưu vào LocalStorage (offline-first).
    *   Nếu người dùng đã đăng nhập, `hoppscotch-backend` sẽ nhận dữ liệu và đồng bộ vào PostgreSQL qua Prisma để người dùng có thể xem lại trên thiết bị khác.
5.  **Giai đoạn Hiển thị (Lenses):** Response trả về được đưa qua các bộ lọc "Lense" (JSON lense, HTML lense, Image lense) để định dạng và hiển thị đẹp mắt cho người dùng.

### Kết luận
Hoppscotch là một minh chứng xuất sắc cho việc kết hợp **TypeScript (NestJS/Vue)** và **Rust (Tauri)** để tạo ra một công cụ vừa có trải nghiệm web tốt, vừa có sức mạnh của ứng dụng bản địa (native). Cách họ sử dụng `fp-ts` cho thấy một tư duy lập trình rất khắt khe về độ ổn định của hệ thống.