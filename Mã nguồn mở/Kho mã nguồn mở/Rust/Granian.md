**Granian** là một HTTP server hiệu năng cao dành cho các ứng dụng Python, được viết bằng **Rust**. Nó được thiết kế để thay thế các giải pháp truyền thống như Gunicorn hay Uvicorn bằng cách tận dụng sức mạnh xử lý I/O của Rust.

Dưới đây là phân tích chi tiết về dự án Granian:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Rust & PyO3:** Granian sử dụng Rust làm ngôn ngữ chủ đạo để đạt được hiệu suất tối đa. **PyO3** đóng vai trò là "cầu nối" (binding) cực kỳ quan trọng, cho phép mã Rust tương tác trực tiếp với các đối tượng Python và trình thông dịch Python (CPython) mà không tốn nhiều chi phí chuyển đổi.
*   **Hyper & Tokio:**
    *   **Hyper:** Một thư viện HTTP (client/server) rất nhanh và chính xác của Rust, chịu trách nhiệm xử lý giao thức HTTP/1.1 và HTTP/2.
    *   **Tokio:** Runtime cho lập trình bất đồng bộ (async), quản lý việc lập lịch các tác vụ (task), xử lý hàng ngàn kết nối đồng thời mà không làm nghẽn hệ thống.
*   **Giao diện hỗ trợ:** Granian hỗ trợ cả 3 tiêu chuẩn chính của Python Web:
    *   **ASGI (Asynchronous Server Gateway Interface):** Tiêu chuẩn hiện đại cho ứng dụng async (FastAPI, Starlette).
    *   **WSGI (Web Server Gateway Interface):** Tiêu chuẩn truyền thống (Flask, Django).
    *   **RSGI (Rust Server Gateway Interface):** Một giao thức riêng do Granian phát triển, tối ưu hóa việc truyền nhận dữ liệu giữa Rust và Python để giảm thiểu overhead của ASGI.

### 2. Tư duy kiến trúc (Architectural Thinking)

*   **Tách biệt I/O và Logic ứng dụng:** Kiến trúc của Granian đẩy toàn bộ phần nặng nề nhất là xử lý kết nối mạng, phân tách gói tin HTTP (parsing), và quản lý SSL/TLS vào mã Rust (I/O threads). Mã Python chỉ tập trung vào việc thực thi logic nghiệp vụ.
*   **Mô hình Worker linh hoạt:**
    *   **MPServer (Multi-Process):** Sử dụng khi chạy với Python có GIL (Global Interpreter Lock) truyền thống. Mỗi worker là một tiến trình riêng biệt để tận dụng đa nhân CPU.
    *   **MTServer (Multi-Thread):** Thiết kế cho các bản build Python **free-threaded** (không có GIL - một tính năng mới từ Python 3.13). Lúc này các worker chạy dưới dạng thread trong cùng một tiến trình, giúp chia sẻ bộ nhớ hiệu quả hơn.
*   **Backpressure (Áp lực ngược):** Granian tích hợp cơ chế quản lý hàng đợi kết nối để tránh tình trạng Rust nhận quá nhiều yêu cầu vượt quá khả năng xử lý của trình thông dịch Python, giúp hệ thống ổn định dưới tải cao.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Async/Await Bridge:** Một kỹ thuật phức tạp trong Granian là việc ánh xạ (mapping) giữa `Future` của Rust (Tokio) và `Coroutine` của Python (AsyncIO). Granian tự xây dựng các `CallbackScheduler` để đảm bảo khi Rust nhận được dữ liệu, nó sẽ kích hoạt đúng task trong event loop của Python.
*   **Zero-copy (Hạn chế sao chép dữ liệu):** Granian cố gắng sử dụng các `Bytes` object để truyền dữ liệu từ lớp HTTP của Rust vào Python mà không cần sao chép lại toàn bộ mảng byte, giúp tiết kiệm CPU và bộ nhớ.
*   **Quản lý GIL thông minh:** Trong các tác vụ chặn (blocking) như WSGI, Granian sử dụng một pool các thread riêng để gọi mã Python, giải phóng thread của Rust để tiếp tục nhận các kết nối mới.
*   **Shared Sockets:** Granian khởi tạo socket ở tiến trình cha và chia sẻ (file descriptor) cho các tiến trình worker con. Điều này cho phép nhiều worker cùng lắng nghe trên một cổng (port) một cách hiệu quả.

### 4. Luồng hoạt động của hệ thống (System Workflow)

1.  **Khởi tạo (Startup):**
    *   Granian CLI nhận cấu hình (host, port, interface...).
    *   Tiến trình chính (Main Process) tạo Shared Socket.
    *   Dựa trên số lượng `--workers`, Granian spawn các tiến trình/thread con.
2.  **Chấp nhận kết nối (Acceptance):**
    *   Mã Rust (Tokio/Hyper) chờ kết nối từ client.
    *   Khi có request, Hyper parse các header HTTP và kiểm tra giao thức.
3.  **Chuyển giao (Dispatch):**
    *   Rust xây dựng một đối tượng `Scope` (chứa thông tin metadata của request).
    *   Rust gọi hàm entry point của ứng dụng Python (đã được load sẵn qua PyO3).
    *   Nếu là ASGI/RSGI, một task sẽ được đẩy vào `asyncio` loop của Python.
4.  **Thực thi & Phản hồi (Execution & Response):**
    *   Ứng dụng Python xử lý logic và gọi hàm `send` (ASGI) hoặc `proto.response` (RSGI).
    *   Lệnh phản hồi này thực chất là gọi ngược lại các phương thức của mã Rust.
    *   Mã Rust nhận dữ liệu từ Python, đóng gói thành response HTTP hoàn chỉnh và gửi trả về client qua socket.
5.  **Dọn dẹp:** Sau khi kết thúc request, các tài nguyên được giải phóng. Nếu kết nối là Keep-alive, socket được giữ lại cho request tiếp theo.

### Kết luận
Granian đại diện cho xu hướng **"Rust-ification"** của hạ tầng Python. Bằng cách thay thế lớp I/O của Python bằng mã Rust cực nhanh, Granian cung cấp một máy chủ có độ trễ thấp và băng thông lớn, đặc biệt vượt trội trong các kịch bản sử dụng HTTP/2 và Websocket.