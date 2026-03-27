Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Trace2Prompt** (một công cụ hỗ trợ debug bằng AI thông qua việc thu thập ngữ cảnh Runtime):

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Ngôn ngữ lập trình:** 
    *   **Backend:** Go (Golang) - Lựa chọn tối ưu để làm một công cụ chạy nền (daemon) vì tính nhẹ, tốn ít RAM và khả năng xử lý đồng thời (concurrency) tốt.
    *   **Frontend Sensor:** Vanilla JavaScript - Được viết dưới dạng một tệp script duy nhất (`trace2prompt.js`) để dễ dàng nhúng vào bất kỳ trang web nào mà không cần build tool phức tạp.
*   **Tiêu chuẩn dữ liệu:** 
    *   **OpenTelemetry (OTLP):** Đây là "xương sống" của hệ thống. Dùng OTLP để nhận Traces, Logs, và Metrics từ các ứng dụng Backend (Java, Python, Node.js...) thông qua Agent mà không cần sửa code.
*   **Giao thức kết nối:** 
    *   **MCP (Model Context Protocol):** Một giao thức mới do Anthropic đề xuất, cho phép các AI IDE (như Cursor) hoặc AI Agent gọi trực tiếp vào Trace2Prompt để lấy ngữ cảnh.
    *   **HTTP/Protobuf:** Dùng để nhận dữ liệu telemetry từ OTel agents.
*   **Cơ sở dữ liệu:** 
    *   **In-memory Storage:** Hệ thống sử dụng các cấu trúc dữ liệu như `map` và `slice` (Ring Buffer) trong RAM để lưu trữ tạm thời các traces gần nhất, đảm bảo tốc độ cực nhanh và không cần cài đặt DB rời.

---

### 2. Tư duy Kiến trúc (Architectural Patterns)

Hệ thống được thiết kế theo mô hình **Sidecar/Collector** thu nhỏ:

*   **Kiến trúc E2E Linking (Nối vết xuyên suốt):** Điểm sáng nhất là khả năng liên kết TraceID từ Frontend (trình duyệt) xuống tận SQL Backend. Nếu Backend không có TraceID, hệ thống sử dụng thuật toán **Fuzzy Matching** (Khớp mờ) dựa trên URL và thời gian (timestamp) để "đoán" các request liên quan.
*   **Kiến trúc "Zero-Config":** Thay vì bắt người dùng sửa code (SDK), dự án tận dụng **Instrumentation** (Agent cho Backend và Monkey Patching cho Frontend).
*   **Kiến trúc Plug-and-Play cho Bảo mật:** Sử dụng cơ chế lọc dữ liệu nhạy cảm (Masking) dựa trên cấu hình YAML. Việc xử lý diễn ra ngay trước khi đóng gói Prompt, giúp bảo vệ dữ liệu người dùng khi gửi lên AI.
*   **Phân tách mối quan tâm (Separation of Concerns):**
    *   `static/trace2prompt.js`: Chuyên trách "do thám" hành vi người dùng trên trình duyệt.
    *   `otel_handlers.go`: Chuyên trách tiếp nhận và phân giải các gói tin OTLP phức tạp.
    *   `prompt_generator.go`: Chuyên trách "biên dịch" dữ liệu thô thành ngôn ngữ tự nhiên cho AI.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Monkey Patching (JS Interceptor):** Trong `trace2prompt.js`, tác giả ghi đè (`override`) các hàm gốc của trình duyệt như `window.fetch`, `XMLHttpRequest.prototype.open`, và `console.log`. Kỹ thuật này cho phép can thiệp vào mọi luồng dữ liệu mà không cần chỉnh sửa code của ứng dụng gốc.
*   **Ring Buffer & Concurrency Control (Go):** 
    *   Sử dụng `sync.Mutex` để đảm bảo an toàn khi nhiều agent gửi dữ liệu cùng lúc.
    *   Cơ chế `MaxBufferSize` giúp kiểm soát bộ nhớ, tự động xóa các traces cũ để tránh tràn RAM.
*   **Regex-based Masking:** Sử dụng các biểu thức chính quy (Regex) được tiền biên dịch (`Pre-compile`) để ẩn các thông tin như Bearer Token, Email, hay Password trong các đoạn text dài hoặc câu lệnh SQL.
*   **Flame Graph Reconstruction:** Từ danh sách các Spans rời rạc gửi về từ OTel, Backend Go thực hiện đệ quy để dựng lại cây thực thi (Execution Tree), giúp AI thấy được thứ tự gọi hàm và thời gian xử lý từng bước.
*   **Embedded Static Files:** Sử dụng tính năng `embed` của Go để đóng gói toàn bộ giao diện Web (HTML/JS) vào trong một tệp thực thi duy nhất (`.exe` hoặc binary), giúp việc triển khai cực kỳ đơn giản (chỉ 1 file).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Thu thập (Collection):**
    *   **Frontend:** Khi người dùng click hoặc gọi API, `trace2prompt.js` tạo ra một TraceID duy nhất, đính kèm vào Header của request và lưu lại hành trình (breadcrumbs).
    *   **Backend:** Agent (ví dụ Java Agent) bắt lấy request, thực thi code, ghi lại các câu SQL và Logs, sau đó gửi gói tin OTLP về cổng `4318`.
2.  **Giai đoạn Xử lý (Processing):**
    *   Trace2Prompt nhận dữ liệu từ cả 2 nguồn. Nó tìm các Spans có cùng TraceID hoặc cùng khung giờ/URL để nhóm lại thành một `TraceRecord`.
    *   Nếu phát hiện lỗi (Status 500 hoặc Exception), hệ thống kích hoạt bộ lọc Masking để che giấu thông tin nhạy cảm.
3.  **Giai đoạn Đóng gói (Prompt Engineering):**
    *   Khi người dùng yêu cầu (qua Web UI hoặc MCP), `generateE2EPrompt` sẽ duyệt qua cây thực thi, trích xuất SQL, Logs, thông tin phần cứng (CPU/RAM) và định dạng chúng thành một bản báo cáo Markdown chuyên nghiệp.
4.  **Giai đoạn Tiêu thụ (Consumption):**
    *   **Cách 1:** Developer copy Prompt từ giao diện web `:4319` dán vào ChatGPT.
    *   **Cách 2:** AI Agent (như Cursor) tự động gọi qua MCP Tool `get_latest_trace` để lấy dữ liệu và tự động sửa code.

### Tổng kết
**Trace2Prompt** là một ví dụ điển hình của việc kết hợp giữa **Observability** (Quan sát hệ thống) và **Generative AI**. Dự án không tập trung vào việc lưu trữ log lâu dài (như ELK stack) mà tập trung vào việc **"tóm tắt ngữ cảnh tức thời"** để biến AI thành một cộng sự debug thực thụ.