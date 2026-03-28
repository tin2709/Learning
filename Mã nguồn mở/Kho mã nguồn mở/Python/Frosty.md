Dưới đây là bản phân tích chi tiết về dự án **Frosty** - một khung làm việc (framework) đa tác nhân (multi-agent) tự vận hành, giúp chuyển đổi ngôn ngữ tự nhiên thành các hoạt động quản trị và truy vấn trên nền tảng dữ liệu Snowflake.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Frosty được xây dựng trên một hệ sinh thái Python hiện đại, tập trung vào khả năng điều phối mô hình ngôn ngữ lớn (LLM):

*   **Google ADK (Agent Development Kit):** Đây là xương sống của dự án, cung cấp cấu trúc để xây dựng hệ thống phân cấp nhiều tác nhân (multi-agent hierarchy), quản lý phiên làm việc (session) và trạng thái (state).
*   **LiteLLM:** Cho phép Frosty trở thành một hệ thống "Bring Your Own Model" (BYOM). Nó hỗ trợ hơn 100 nhà cung cấp model (OpenAI, Anthropic, Gemini) thông qua một giao diện thống nhất.
*   **Snowflake Ecosystem:**
    *   `snowflake-snowpark-python`: Dùng cho các xử lý dữ liệu phức tạp.
    *   `snowflake-connector-python`: Kết nối tiêu chuẩn để thực thi SQL/DDL.
*   **Giao diện dòng lệnh (CLI):** Sử dụng `Rich` (để render Markdown, bảng biểu, màu sắc) và `prompt_toolkit` (để quản lý input người dùng chuyên nghiệp với khung viền và nhắc lệnh).
*   **Observability:** Tích hợp sẵn **OpenTelemetry** để đẩy traces/metrics về Grafana Cloud, giúp giám sát hiệu năng của từng Agent.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Frosty đi theo mô hình **Phân rã chuyên môn (Specialized Decomposition)**:

*   **Hệ thống 153 Agent chuyên biệt:** Thay vì dùng một Prompt khổng lồ, Frosty chia nhỏ Snowflake thành 7 "Trụ cột" (Pillars) như Data Engineer, Security, Governance, Inspector... Mỗi trụ cột chứa các "Chuyên gia" (Specialists) chỉ lo một việc duy nhất (ví dụ: Agent chỉ chuyên về Masking Policy).
*   **Mô hình Manager-Worker:** 
    *   `CLOUD_DATA_ARCHITECT` đóng vai trò quản lý cấp cao, phân tích ý định người dùng, lập kế hoạch thực thi (plan) và điều phối công việc cho các trụ cột.
    *   Các trụ cột tiếp tục ủy quyền cho các chuyên gia thực thi.
*   **Kiến trúc Lazy Loading:** Để tránh việc tốn tài nguyên và thời gian khởi động, Frosty không import tất cả agent cùng lúc. Nó sử dụng `LazyAgentTool` để import module chỉ khi agent đó thực sự được gọi. Một luồng chạy ngầm sẽ "làm ấm" (pre-warm) các agent theo thứ tự ưu tiên (BFS).
*   **Local-First & Privacy:** Toàn bộ logic chạy trên môi trường của người dùng. Thông tin định danh (credentials) không bao giờ rời khỏi máy cục bộ, chỉ có các prompt và cấu trúc dữ liệu được gửi đến LLM.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

Dự án áp dụng nhiều kỹ thuật nâng cao để đảm bảo tính chính xác và an toàn:

*   **Safety Gate (Cổng an toàn 2 lớp):**
    *   *Lớp Prompt:* Cấu hình Agent luôn ưu tiên `CREATE IF NOT EXISTS` thay vì `CREATE OR REPLACE`.
    *   *Lớp Code (Hard-coded):* Interceptor trong `tools.py` quét SQL. Lệnh `DROP` bị chặn hoàn toàn ở mức mã nguồn. Lệnh `CREATE OR REPLACE` bị tạm dừng để yêu cầu người dùng xác nhận thủ công (Human-in-the-loop).
*   **Skill Injection (Hệ thống Kỹ năng):** Sử dụng các file `SKILL.md` và `parameters.md` trong thư mục `skills/`. Đây là kỹ thuật RAG (Retrieval-Augmented Generation) mức độ nhẹ, cung cấp tài liệu hướng dẫn chuẩn của Snowflake cho Agent ngay khi thực thi để tránh ảo giác (hallucination).
*   **Dry Run & Validation:** Đối với các đối tượng phức tạp như Stored Procedure, Frosty thực hiện quy trình 2 bước: tạo một procedure tạm với tên ngẫu nhiên trong một Transaction, chạy thử, sau đó Rollback. Chỉ khi bước này thành công, procedure thật mới được tạo.
*   **State Scoping:** Quản lý trạng thái theo phạm vi:
    *   `user:` (vĩnh viễn cho người dùng).
    *   `app:` (chia sẻ trong toàn bộ phiên làm việc của ứng dụng).
    *   `temp:` (chỉ dùng cho một lượt tương tác).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình xử lý một yêu cầu diễn ra như sau:

1.  **Tiếp nhận (Input):** Người dùng nhập: *"Set up MFA cho các user chưa có"*.
2.  **Phân tích & Thăm dò (Intent & Inspection):**
    *   Manager nhận yêu cầu.
    *   Gọi `INSPECTOR_PILLAR` để quét danh sách user hiện tại trong Snowflake (không đoán mò).
3.  **Lập kế hoạch (Planning):** Manager tạo danh sách các bước: (1) Liệt kê user thiếu MFA, (2) Tạo lệnh `ALTER USER` cho từng người, (3) Thực thi.
4.  **Ủy quyền (Delegation):** Manager chuyển nhiệm vụ (2) và (3) cho `ADMINISTRATOR` pillar, trụ cột này gọi chuyên gia `User Agent`.
5.  **Thực thi an toàn (Execution):** 
    *   Agent tạo SQL.
    *   SQL đi qua `execute_query` safety gate.
    *   Hệ thống yêu cầu người dùng gõ `yes` trên terminal.
    *   Gửi lệnh đến Snowflake.
6.  **Xác nhận (Validation):** Manager kiểm tra trạng thái session thông qua `app:TASKS_PERFORMED` để đảm bảo bước trước đã thành công rồi mới đi tiếp.
7.  **Kết xuất (Output):** Hiển thị kết quả bằng bảng Markdown và lưu lại toàn bộ lịch sử truy vấn vào file `.sql`.

### Tổng kết
Frosty không chỉ là một công cụ AI đơn thuần, nó là một **Hệ điều hành bằng ngôn ngữ tự nhiên cho Snowflake**. Sự kết hợp giữa việc quản lý agent phân cấp chặt chẽ và lớp bảo mật cứng bằng mã nguồn khiến nó trở nên khả thi cho các môi trường doanh nghiệp thực tế.