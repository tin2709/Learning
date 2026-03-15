Dưới đây là phân tích chi tiết về dự án **catpaw** - một Agent giám sát thông minh thế hệ mới, kết hợp giữa giám sát truyền thống và trí tuệ nhân tạo (AI).

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Go (Golang)** chiếm 99.8%. Lựa chọn này giúp tạo ra một file thực thi duy nhất (single binary), không phụ thuộc (zero dependencies), cực kỳ nhẹ và phù hợp để chạy trên nhiều hệ điều hành (Linux, Windows, macOS).
*   **Mô hình ngôn ngữ lớn (LLM):** Hỗ trợ đa dạng các đầu vào từ OpenAI, AWS Bedrock và các Gateway AI khác. Dự án có cơ chế **Failover** (tự động chuyển đổi model nếu lỗi) và **Retry** để đảm bảo chẩn đoán không bị gián đoạn.
*   **Giao thức MCP (Model Context Protocol):** Đây là điểm nhấn hiện đại, cho phép AI kết nối trực tiếp với các nguồn dữ liệu bên ngoài như Prometheus, Jaeger, CMDB để lấy thêm ngữ cảnh khi chẩn đoán.
*   **Cấu hình:** Sử dụng định dạng **TOML** cho phép cấu hình phân cấp, hỗ trợ gom nhóm (partials) để tái sử dụng cấu hình.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của catpaw được thiết kế theo mô hình **"Plugin-based & AI-RCA (Root Cause Analysis)"**:

*   **Tính module hóa tuyệt đối:** Mỗi plugin kiểm tra (Check Plugins) và mỗi công cụ chẩn đoán (Diagnose Tools) là một package độc lập. Điều này giúp giảm thiểu "context" mà AI cần xử lý và dễ dàng mở rộng.
*   **Sự tách biệt giữa "Giám sát" và "Chẩn đoán":** 
    *   *Monitoring:* Chỉ tập trung phát hiện bất thường (Anomalies).
    *   *AI Diagnose:* Chỉ kích hoạt khi có sự cố hoặc yêu cầu từ người dùng, tránh lãng phí tài nguyên LLM.
*   **Thiết kế hướng sự kiện (Event-driven):** Các plugin sản sinh ra các sự kiện (Events) chuẩn hóa. Các sự kiện này đi qua một Engine để xử lý (khử trùng, ức chế) trước khi quyết định gửi thông báo hoặc gọi AI.
*   **Cơ chế In-flight & Cooldown:** Để tránh việc AI chẩn đoán lặp đi lặp lại cho cùng một lỗi trong thời gian ngắn (gây tốn token), hệ thống có cơ chế hàng chờ và thời gian chờ (cooldown).

### 3. Các kỹ thuật chính (Key Techniques)

*   **AI Tool Calling (Function Calling):** Đây là kỹ thuật cốt lõi. AI không "đoán" nguyên nhân mà được cấp quyền gọi hơn 70 công cụ hệ thống (như `top`, `df`, `ss`, `dmesg`, `oom_history`). AI sẽ tự quyết định chạy lệnh nào dựa trên triệu chứng của lỗi.
*   **Cảm biến "Silent Killers":** Tập trung vào các lỗi "ngầm" của Linux mà các công cụ thông thường hay bỏ qua như: tràn bảng conntrack, tràn bảng lân cận ARP (neigh), trôi thông số sysctl, hoặc tràn hàng đợi listen socket.
*   **Interactive Chat REPL:** Cung cấp lệnh `catpaw chat`, biến terminal thành một giao diện trò chuyện. Người dùng có thể hỏi "Tại sao CPU cao?" và AI sẽ tự thực hiện các lệnh kiểm tra và trả lời bằng ngôn ngữ tự nhiên.
*   **Hệ thống Notifier đa hướng:** Hỗ trợ cùng lúc nhiều kênh thông báo (Console, Flashduty, PagerDuty, WebAPI) để đảm bảo thông tin chẩn đoán đến đúng người dùng.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Quy trình hoạt động tự động của catpaw diễn ra như sau:

1.  **Giai đoạn Giám sát (Detection):** Các Check Plugins (như `cpu`, `disk`, `redis`) chạy định kỳ theo cấu hình. Nếu phát hiện vượt ngưỡng, một `Event` với trạng thái `Warning` hoặc `Critical` được tạo ra.
2.  **Giai đoạn Thông báo (Alerting):** `Engine` nhận Event, chuyển đến các `Notifiers`. Người dùng nhận được cảnh báo ngay lập tức.
3.  **Giai đoạn Kích hoạt AI (Trigger):** Nếu cấu hình AI được bật, `DiagnoseAggregator` sẽ gom các lỗi liên quan trong một cửa sổ thời gian (ví dụ 5 giây) và gửi yêu cầu chẩn đoán đến `DiagnoseEngine`.
4.  **Giai đoạn Chẩn đoán (RCA):** 
    *   AI nhận ngữ cảnh lỗi.
    *   AI gọi các `Diagnose Tools` thích hợp để lấy dữ liệu thực tế từ hệ thống.
    *   AI thực hiện nhiều vòng suy luận (Max Rounds).
5.  **Giai đoạn Báo cáo (Reporting):** Một báo cáo chẩn đoán bằng Markdown (gồm: tóm tắt, phân tích nguyên nhân gốc, đề xuất xử lý) được tạo ra và gửi ngược lại các kênh thông báo như một Event bổ sung.

### Kết luận
**catpaw** không chỉ là một công cụ giám sát; nó đóng vai trò như một **SRE ảo** (Virtual SRE) túc trực ngay trên máy chủ. Điểm mạnh nhất của nó là khả năng chuyển hóa từ một cảnh báo khô khan sang một báo cáo có thể hành động ngay lập tức (actionable insight) nhờ sức mạnh của AI và bộ công cụ chẩn đoán sâu.