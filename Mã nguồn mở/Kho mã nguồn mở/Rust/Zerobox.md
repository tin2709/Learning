Dựa trên mã nguồn và tài liệu của dự án **Zerobox**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)

Zerobox là một giải pháp sandbox (hộp cát) đa nền tảng, tập trung vào hiệu suất và bảo mật cho việc thực thi các tiến trình không tin cậy (đặc biệt là mã do AI tạo ra).

*   **Ngôn ngữ lập trình:** **Rust** (chiếm ~63%) cho lõi hệ thống để đảm bảo an toàn bộ nhớ và tốc độ; **TypeScript** (~26%) cho bộ SDK dành cho các nhà phát triển Node.js.
*   **Công nghệ Sandbox (Hệ điều hành):**
    *   **Linux:** Sử dụng **Bubblewrap** (namespaces), **Seccomp** (lọc system calls) và **Landlock** (kiểm soát truy cập file cấp nhân).
    *   **macOS:** Sử dụng **Seatbelt** (`sandbox-exec`), cơ chế bảo mật nội tại của Apple.
    *   **Windows:** Sử dụng Restricted Tokens và ACLs (đang trong lộ trình phát triển).
*   **Networking & Proxy:** Sử dụng framework **Rama** (tương tự Tower/Axum) để xây dựng một proxy mạng siêu nhẹ, hỗ trợ lọc tên miền và tấn công giả mạo có chủ đích (MITM) để tiêm thông tin xác thực.
*   **Runtime:** **Tokio** xử lý các tác vụ bất đồng bộ cho cả tiến trình chính và proxy mạng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Zerobox được thiết kế dựa trên ba trụ cột chính:

*   **Triết lý "Từ chối theo mặc định" (Deny-by-default):** Không giống như các tiến trình thông thường có quyền truy cập rộng rãi, Zerobox chặn toàn bộ quyền ghi (write), mạng (network) và các biến môi trường (env vars) trừ khi người dùng cấp phép rõ ràng.
*   **Cơ chế "Vault" cho thông tin xác thực:** Đây là điểm sáng tạo nhất. Thay vì đưa API Key thật vào biến môi trường (nơi mã độc có thể đọc và đánh cắp), Zerobox đưa một **placeholder** (mã giữ chỗ ngẫu nhiên). API Key thật chỉ tồn tại trong bộ nhớ của Proxy.
*   **Tách biệt lớp thực thi và lớp SDK:** Lõi Rust (`crates/zerobox`) xử lý logic hệ thống thấp cấp, trong khi SDK TypeScript (`packages/zerobox`) cung cấp giao diện lập trình dễ tiếp cận (Deno-style API) cho cộng đồng Web.
*   **Sự phụ thuộc vào OpenAI Codex:** Dự án tận dụng (vendor) các thành phần sandboxing mạnh mẽ từ `codex-rs` của OpenAI, đảm bảo độ tin cậy tương đương với các hệ thống chạy mã tự động quy mô lớn.

---

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **MITM Proxy Substitution (Tiêm bí mật qua Proxy):** Khi tiến trình con gửi một request HTTP có chứa placeholder bí mật trong Header, Proxy sẽ chặn request đó, thay placeholder bằng Key thật rồi mới gửi đi. Kỹ thuật này triệt tiêu nguy cơ rò rỉ Key từ bộ nhớ của tiến trình bị sandbox.
*   **Arg0 Dispatching:** Zerobox sử dụng kỹ thuật đa nhiệm trong một file binary duy nhất. Tùy thuộc vào tên gọi (argv[0]), nó có thể đóng vai trò là CLI người dùng hoặc là một "helper" chạy bên trong sandbox để thực hiện các thiết lập cuối cùng.
*   **Environment Cleaning:** Hệ thống thực hiện một bước "quét sạch" môi trường, chỉ giữ lại các biến thiết yếu (PATH, HOME, TERM...) để tránh việc rò rỉ dữ liệu nhạy cảm của máy chủ thông qua các biến môi trường không tên.
*   **Path Mapping & Resolution:** Sử dụng thư viện `codex-utils-absolute-path` để giải quyết các đường dẫn tương đối thành tuyệt đối trước khi áp dụng chính sách bảo mật, tránh các lỗi leo thang đặc quyền qua symlink.

---

### 4. Luồng hoạt động hệ thống (Operational Workflow)

Một chu kỳ chạy của Zerobox diễn ra qua các bước sau:

1.  **Tiếp nhận chính sách (Policy Parsing):** CLI hoặc SDK nhận các cờ như `--allow-read`, `--allow-net`, `--secret`.
2.  **Khởi tạo Proxy (Nếu cần):** Nếu có yêu cầu lọc mạng hoặc dùng bí mật, Zerobox khởi động một proxy cục bộ và tạo một chứng chỉ CA tạm thời để tiến trình con tin tưởng (MITM).
3.  **Chuẩn bị môi trường:** 
    *   Tạo các placeholder ngẫu nhiên cho các `--secret`.
    *   Xác định các vùng file hệ thống cần thiết (libs, binaries) và vùng người dùng cho phép.
4.  **Thiết lập rào chắn (Sandbox Isolation):**
    *   **Trên Linux:** Gọi Bubblewrap để tạo các namespaces mới (User, PID, Network).
    *   **Trên macOS:** Tạo profile Seatbelt động và thực thi qua `sandbox-exec`.
5.  **Thực thi tiến trình con:** Lệnh mục tiêu (ví dụ: `node app.js`) được chạy trong môi trường đã bị cô lập.
6.  **Điều phối thời gian thực:** 
    *   Tiến trình con gọi mạng -> Đi qua Proxy -> Proxy kiểm tra tên miền (allow/deny) -> Proxy thay placeholder bằng API Key thật.
    *   Tiến trình con cố gắng ghi file ngoài danh sách -> Hệ điều hành chặn lại dựa trên chính sách Landlock/Seatbelt.
7.  **Thu hồi kết quả:** Sau khi tiến trình con kết thúc, Zerobox dọn dẹp proxy và trả về output cho tiến trình cha.

### Kết luận
Zerobox không cố gắng thay thế Docker hay VM, mà nó lấp đầy khoảng trống cho các tác vụ cần **cô lập cực nhanh (~10ms overhead)** và **quản lý bí mật an toàn**. Đây là công cụ lý tưởng cho các AI Agent cần chạy code Python/JS từ Internet mà không sợ hỏng hệ thống hay mất cắp API Key.