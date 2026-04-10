Dựa trên các tệp mã nguồn và cấu trúc thư mục của dự án **Lodestone**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Lodestone là một hệ thống quản lý máy chủ game (Minecraft và các game khác) hiện đại, sử dụng mô hình **Client-Server** nhưng có thể đóng gói thành ứng dụng **Desktop** duy nhất.

*   **Backend (Core):**
    *   **Ngôn ngữ:** Rust (đảm bảo an toàn bộ nhớ và hiệu suất cao). Dự án sử dụng `#![forbid(unsafe_code)]` để tối đa hóa độ tin cậy.
    *   **Web Framework:** **Axum** (dựa trên Tokio) để xây dựng REST API và WebSocket.
    *   **Runtime cho Macro:** Tích hợp **Deno** (`deno_core`, `deno_runtime`) để cho phép người dùng viết các bản mở rộng (macros) bằng TypeScript/JavaScript chạy trong môi trường sandbox an toàn.
    *   **Cơ sở dữ liệu:** **SQLite** (thông qua `sqlx`) để lưu trữ cấu hình người dùng, sự kiện (events) và logs.
    *   **Hệ thống bất đồng bộ:** **Tokio** là nền tảng xử lý đồng thời.
    *   **Tích hợp Docker:** Sử dụng crate `bollard` để quản lý các container trực tiếp từ Rust.
    *   **Networking:** Tích hợp **playit-gg** để hỗ trợ kết nối không cần mở port (port forward).

*   **Frontend (Dashboard):**
    *   **Framework:** **Next.js (React)** + **TypeScript**.
    *   **Desktop Wrapper:** **Tauri** (sử dụng Rust để bắc cầu giữa giao diện web và các tính năng hệ thống của máy tính).
    *   **Styling:** **Tailwind CSS**.
    *   **Quản lý trạng thái & Data Fetching:** **TanStack Query** (React Query).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế theo hướng **Modular** và **Trait-based**:

*   **Trừu tượng hóa đối tượng máy chủ (Trait-based Design):**
    *   Lodestone định nghĩa các **Traits** trong `core/src/traits/` như `TServer`, `TConfigurable`, `TPlayerManagement`.
    *   Bất kỳ loại game nào (Minecraft Java, Bedrock, hay một ứng dụng Generic) chỉ cần cài đặt các Trait này là có thể tích hợp vào hệ thống.
    *   Sử dụng crate `enum_dispatch` để gọi phương thức trên các kiểu instance khác nhau mà không bị ảnh hưởng bởi overhead của Dynamic Dispatch (vtable).

*   **Kiến trúc hướng sự kiện (Event-Driven Architecture):**
    *   Mọi thay đổi trạng thái (server chạy, người dùng join, log mới) đều được đẩy vào một `EventBroadcaster` (sử dụng broadcast channel của Tokio).
    *   Các sự kiện này được lưu vào cơ sở dữ liệu và đồng thời đẩy về phía Dashboard thông qua **WebSockets** để cập nhật giao diện thời gian thực.

*   **Kiến trúc Sandbox (Secure Scripting):**
    *   Thay vì cho phép chạy script trực tiếp trên hệ thống, dự án nhúng toàn bộ runtime của Deno vào Core. Điều này cho phép tạo ra các Macro có thể tương tác với máy chủ game nhưng bị giới hạn quyền truy cập file/mạng thông qua hệ thống `deno_ops` tùy chỉnh.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Bắc cầu ngôn ngữ (FFI & Ops):**
    *   Trong `core/src/deno_ops/`, các lập trình viên Rust viết các hàm (ops) để JavaScript trong Macro có thể gọi ngược lại logic của Rust (ví dụ: `send_command`, `get_instance_state`).
    *   Sử dụng `ts-rs` để tự động tạo các kiểu dữ liệu TypeScript từ cấu trúc Rust, đảm bảo tính đồng bộ kiểu (Type safety) giữa Backend và Frontend/Macros.

*   **Xử lý luồng dữ liệu (Streaming):**
    *   Sử dụng `BufReader` và `AsyncBufReadExt` để đọc output từ tiến trình máy chủ game (stdout/stderr) theo thời gian thực và phân tích cú pháp bằng Regex để nhận diện sự kiện (ví dụ: người chơi join game).

*   **Quản lý vòng đời tiến trình (Process Management):**
    *   Sử dụng `Arc<Mutex<Option<Child>>>` để quản lý các tiến trình máy chủ đang chạy ngầm, cho phép khởi động, dừng, hoặc kill tiến trình một cách an toàn từ nhiều luồng.

*   **Tự động hóa Migration:**
    *   Hệ thống có logic tự động nâng cấp cấu hình cũ (v0.4.x) lên cấu hình mới (v0.5.x) trong thư mục `core/src/migration/`, đảm bảo tính tương thích ngược khi người dùng cập nhật phần mềm.

### 4. Luồng hoạt động hệ thống (System Operation Flows)

#### A. Luồng tạo Máy chủ (Instance Creation):
1.  **Dashboard:** Người dùng chọn loại game và nhập cấu hình.
2.  **Core (Axum API):** Nhận yêu cầu, tạo cấu trúc thư mục trong `instances/`.
3.  **Download:** Core tự động tìm URL JRE (Java Runtime) và file Jar của game (Vanilla, Fabric, Forge...) dựa trên phiên bản người dùng chọn, thực hiện tải và giải nén.
4.  **Config:** Tạo file `.lodestone_config` để đánh dấu thư mục là một instance hợp lệ.

#### B. Luồng thực thi Macro (Macro Execution):
1.  **Trigger:** Người dùng nhấn "Run Macro" hoặc một sự kiện kích hoạt.
2.  **Deno Worker:** `MacroExecutor` khởi tạo một Deno Worker mới.
3.  **Injection:** Rust inject các biến môi trường (như `__instance_uuid`) và các `ops` vào runtime.
4.  **Running:** Code TypeScript của người dùng chạy, gọi các hàm API của Lodestone để tự động hóa tác vụ (ví dụ: tự động backup khi server vắng người).

#### C. Luồng Đồng bộ hóa trạng thái (Real-time Sync):
1.  **Monitor:** Core chạy một task nền dùng `sysinfo` để lấy CPU/RAM của server game mỗi giây.
2.  **Broadcast:** Dữ liệu performance + logs được đẩy vào `event_broadcaster`.
3.  **Frontend:** Dashboard đang kết nối WebSocket nhận được gói tin JSON, dùng React Query để cập nhật biểu đồ và bảng console ngay lập tức mà không cần load lại trang.

### Tổng kết
Lodestone là một dự án có độ phức tạp kỹ thuật cao, kết hợp khéo léo giữa tính an toàn của **Rust**, khả năng mở rộng của **Deno** và sự linh hoạt của **React**. Tư duy thiết kế "mọi thứ là một instance" giúp nó dễ dàng mở rộng cho các loại server game khác ngoài Minecraft trong tương lai.