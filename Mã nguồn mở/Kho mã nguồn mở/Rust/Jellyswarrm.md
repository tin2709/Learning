Dựa trên mã nguồn bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Jellyswarrm** - một giải pháp Reverse Proxy mạnh mẽ dùng để hợp nhất nhiều server Jellyfin.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng dựa trên hệ sinh thái **Rust** hiện đại, ưu tiên hiệu năng và an toàn bộ nhớ:

*   **Backend Framework:** Sử dụng **Axum** (dựa trên Hyper và Tokio). Đây là web framework có tốc độ cực cao, hỗ trợ tốt cho việc xử lý bất đồng bộ và các kết nối duy trì lâu (như streaming).
*   **Async Runtime:** **Tokio** là nền tảng cho việc xử lý đồng thời (concurrency).
*   **Database & Migration:** **SQLx** với **SQLite**. Dự án sử dụng SQLite để lưu trữ thông tin mapping người dùng, mapping media ID và các session mà không cần một hệ quản trị DB phức tạp.
*   **Template Engine:** **Askama**. Đây là một template engine biên dịch (compile-time), giúp kiểm tra lỗi ngay khi build và có tốc độ render cực nhanh cho giao diện quản lý.
*   **Networking:** **Reqwest** được dùng để gửi các request từ Proxy đến các server Jellyfin upstream.
*   **Serialization:** **Serde** đóng vai trò quan trọng trong việc chuyển đổi qua lại các cấu trúc dữ liệu JSON phức tạp giữa client và server.
*   **Frontend (Management UI):** Sử dụng **HTMX** kết hợp với CSS framework **Pico.css**. Điều này cho phép tạo ra trải nghiệm "Single Page Application" mượt mà mà không cần các framework JS nặng nề như React/Vue.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Jellyswarrm không chỉ đơn thuần là chuyển tiếp request, mà nó hoạt động như một **Virtual Aggregator**:

*   **Virtual ID Mapping (Cơ chế ánh xạ ID ảo):** Đây là tư duy cốt lõi. Mỗi bộ phim/tập phim từ server gốc sẽ có một ID riêng. Jellyswarrm tạo ra một "Virtual ID" duy nhất lưu trong database. Khi client yêu cầu một ID ảo, proxy sẽ tra cứu xem ID đó thuộc server nào và ID gốc là gì để điều phối request chính xác.
*   **Identity Federation (Liên hợp định danh):** Proxy cho phép một tài khoản Jellyswarrm duy nhất được ánh xạ tới nhiều tài khoản trên các server Jellyfin khác nhau. Nó quản lý thông tin đăng nhập và tự động thực hiện xác thực với từng server upstream.
*   **Parallel Fan-out:** Khi client yêu cầu danh sách phim "Mới nhất", proxy thực hiện gửi request song song đến *tất cả* các server đã kết nối, sau đó tổng hợp, sắp xếp (interleave) và trả về kết quả hợp nhất.
*   **Cấu trúc Đa tầng (Multi-crate Workspace):**
    *   `jellyfin-api`: Thư viện giao tiếp chuẩn với Jellyfin.
    *   `jellyswarrm-macros`: Chứa các procedural macros để xử lý sự không đồng nhất trong chuẩn đặt tên (PascalCase vs camelCase) của Jellyfin API.
    *   `jellyswarrm-proxy`: Core logic xử lý routing và mapping.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Procedural Macros:** Sử dụng macro `#[multi_case_struct]` để tự động tạo ra các thuộc tính `#[serde(alias = "...")]`. Kỹ thuật này giải quyết vấn đề lớn của Jellyfin là API trả về kết quả lúc thì chữ hoa đầu (Pascal), lúc thì chữ thường đầu (camel).
*   **Async Recursion:** Trong `json_processor.rs`, dự án sử dụng kỹ thuật đệ quy bất đồng bộ để duyệt qua toàn bộ cây JSON của response, tìm kiếm các trường ID và thay thế chúng bằng Virtual ID trước khi trả về cho client.
*   **Streaming Middleware:** Proxy xử lý dữ liệu video/audio dưới dạng stream. Kỹ thuật `hyper::Body` kết hợp với `reqwest::Response::bytes_stream` giúp proxy chuyển tiếp dữ liệu mà không cần nạp toàn bộ tệp vào RAM, giảm độ trễ (latency).
*   **Health Check Loop:** Một vòng lặp chạy ngầm (background task) định kỳ kiểm tra trạng thái sống/chết của các server upstream để cập nhật priority và trạng thái hiển thị trên UI.
*   **Centralized Context:** Sử dụng `AppState` bao bọc trong `Arc` để chia sẻ trạng thái giữa các handler của Axum một cách an toàn giữa các luồng.

---

### 4. Luồng hoạt động hệ thống (System Flow)

#### A. Luồng Đăng nhập (Authentication Flow):
1.  Người dùng gửi thông tin đăng nhập đến Jellyswarrm.
2.  Jellyswarrm thử đăng nhập vào các server Jellyfin đã mapping.
3.  Nếu thành công, nó tạo ra một `AuthorizationSession` nội bộ, trả về một token ảo cho client. Token này thực chất là chìa khóa để proxy thay mặt người dùng giao tiếp với các server gốc.

#### B. Luồng Truy vấn Media (Item Retrieval Flow):
1.  Client gửi request (ví dụ: lấy danh sách phim).
2.  Proxy phân tích request, xác định người dùng là ai.
3.  Proxy gửi request đồng thời (parallel) đến các server mà người dùng có quyền truy cập.
4.  **Xử lý dữ liệu:** Proxy nhận JSON từ các server, duyệt qua JSON để thay thế tất cả ID thật bằng ID ảo, đồng thời đính kèm tên server vào tiêu đề phim (ví dụ: "Inception [Server-A]").
5.  Proxy trộn kết quả và trả về cho client.

#### C. Luồng Phát video (Streaming Flow):
1.  Client gửi yêu cầu phát video dựa trên ID ảo.
2.  Proxy tra cứu database để tìm server đích và ID thật.
3.  **Quyết định chế độ phát:**
    *   **Redirect:** Proxy trả về mã HTTP 302, đẩy client kết nối trực tiếp đến server gốc (tiết kiệm băng thông cho proxy).
    *   **Proxy:** Proxy đứng giữa làm "ống dẫn" dữ liệu, hữu ích khi client không thể kết nối trực tiếp đến server gốc (qua NAT/Firewall).

### Tổng kết
Jellyswarrm là một dự án có kỹ thuật lập trình Rust rất tốt, đặc biệt là trong việc xử lý dữ liệu JSON động và tối ưu hóa luồng I/O. Nó giải quyết bài toán trải nghiệm người dùng phân tán một cách thông minh bằng cách sử dụng lớp ánh xạ ID ảo.