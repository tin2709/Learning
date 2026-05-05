Dựa trên phân tích mã nguồn và tài liệu kiến trúc của **Inngest**, đây là bản phân tích chuyên sâu về hệ thống điều phối workflow (workflow orchestration) hiện đại này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình:** 
    *   **Go (Backend):** Chiếm ~59%, dùng cho lõi hệ thống (Runner, Executor, Queue) nhờ khả năng xử lý đồng thời (concurrency) cực tốt.
    *   **TypeScript (UI/SDK):** Chiếm ~39%, dùng cho Dashboard và Inngest JS SDK.
    *   **Lua:** Dùng để viết các script thực thi nguyên tử (atomic) bên trong Redis để quản lý hàng đợi và lock.
*   **Giao tiếp & Định nghĩa dữ liệu:**
    *   **Protobuf & gRPC (ConnectRPC):** Inngest sử dụng Protobuf làm "nguồn sự thật" duy nhất cho dữ liệu, giúp đồng bộ hóa kiểu dữ liệu giữa Go backend và các SDK.
    *   **GraphQL & REST (V2):** Cung cấp API cho Dashboard và tích hợp bên thứ ba.
*   **Lưu trữ & Persistence:**
    *   **PostgreSQL:** Cơ sở dữ liệu chính lưu trữ cấu thực thể hệ thống (Apps, Functions, Events).
    *   **Redis/Valkey/Garnet:** Dùng làm State Store và Queue layer cho các tác vụ thời gian thực.
    *   **SQLite:** Dùng cho Dev Server để người dùng chạy local không cần cài đặt phức tạp.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Inngest được xây dựng xung quanh khái niệm **Durable Execution** (Thực thi bền bỉ):

*   **Event-Driven (Hướng sự kiện):** Hệ thống không chờ đợi phản hồi trực tiếp. Khi một sự kiện đến, nó được đưa vào Event Stream, Runner sẽ quyết định function nào cần chạy dựa trên trigger.
*   **Separation of Concerns (Phân tách trách nhiệm):**
    *   **Event API:** Chỉ nhận và xác thực sự kiện.
    *   **Runner:** Logic lập lịch, kiểm tra điều kiện trigger và quản lý trạng thái Run.
    *   **Executor:** Phụ trách gọi code người dùng, xử lý retry và lưu lại kết quả từng bước (step).
*   **Multi-tenancy & Sharding:** Thiết kế hàng đợi hỗ trợ phân mảnh (sharding) và phân vùng (partitioning) để đảm bảo tính công bằng (fairness) giữa các user và khả năng mở rộng cực lớn.
*   **Local-First DX:** Dev Server được thiết kế để giả lập môi trường Cloud hoàn hảo, giúp lập trình viên debug workflow ngay trên máy cá nhân thông qua giao diện trực quan.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **State Memoization (Ghi nhớ trạng thái):** Đây là kỹ thuật quan trọng nhất. Inngest không lưu toàn bộ code, nó lưu kết quả của từng `step`. Khi một function bị crash hoặc restart, nó "chạy lại" nhưng các step đã xong sẽ được lấy từ State Store thay vì thực thi lại logic bên trong.
*   **Connect Protocol (WebSocket-based):** Sử dụng WebSocket để duy trì kết nối hai chiều giữa Inngest Server và SDK người dùng. Điều này cho phép server "đẩy" lệnh thực thi xuống SDK ngay cả khi SDK nằm sau tường lửa (NAT).
*   **Atomic Lua Scripting:** Các thao tác phức tạp trên hàng đợi (như cấp phát lease cho shard, quản lý semaphore cho concurrency) được viết bằng Lua để chạy trực tiếp trên Redis, đảm bảo tính nhất quán dữ liệu mà không bị race condition.
*   **Code Generation:** Sử dụng `sqlc` để tạo code Go từ SQL, `buf` cho Protobuf, và `tygo` để chuyển đổi struct Go sang interface TypeScript cho UI. Điều này giảm thiểu lỗi do sai lệch kiểu dữ liệu giữa các tầng.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một "Durable Function" trong Inngest:

1.  **Ingestion:** SDK gửi một Event (JSON) kèm theo Event Key qua HTTP.
2.  **Triggering:** Runner nhận Event -> Tra cứu bảng đăng ký Function -> Nếu khớp điều kiện (trigger/expression) -> Tạo một **Function Run ID**.
3.  **Queuing:** Run ID được đẩy vào Queue layer. Tại đây, các bộ lọc **Flow Control** (Throttling, Debounce, Concurrency) sẽ quyết định khi nào thì Run này được phép thực thi.
4.  **Execution (The Loop):**
    *   Executor lấy task từ hàng đợi.
    *   Executor gửi yêu cầu thực thi kèm theo lịch sử các step (nếu có) đến SDK người dùng qua HTTP hoặc WebSocket.
    *   SDK chạy code đến `step` đầu tiên chưa có kết quả -> Trả kết quả step đó về Executor.
    *   Executor lưu kết quả vào **State Store** và đẩy lại task vào hàng đợi để chạy step kế tiếp.
5.  **Completion:** Khi toàn bộ code function chạy xong, Executor đánh dấu Run là `Completed`, lưu output cuối cùng và kích hoạt các event hệ thống (nếu có).

### Tổng kết
Inngest chuyển đổi sự phức tạp của hạ tầng (retry, queue, state) thành code ứng dụng đơn giản. Sự kết hợp giữa **Go (hiệu năng)** và **Protobuf (chặt chẽ)** giúp họ tạo ra một hệ thống điều phối workflow có độ tin cậy cao, có khả năng chạy các quy trình kéo dài hàng tháng mà không mất dữ liệu.