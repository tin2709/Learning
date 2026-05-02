Dựa trên cấu trúc mã nguồn và tài liệu kỹ thuật của dự án **HASH** (từ HashIntel), dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và cơ chế hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

HASH là một dự án "Polyglot" (đa ngôn ngữ) quy mô lớn, kết hợp giữa hiệu năng của Rust và sự linh hoạt của TypeScript:

*   **Ngôn ngữ lập trình:**
    *   **Rust (52%):** Được dùng để xây dựng nhân hệ thống (Core Engine), lớp truy vấn đồ thị (`hash-graph`), trình biên dịch ngôn ngữ truy vấn tùy chỉnh (`hashql`), và các thư viện xử lý dữ liệu cấp thấp.
    *   **TypeScript (47%):** Dùng cho toàn bộ Frontend (Next.js), API trung gian (Node.js), và các Worker điều phối AI (Temporal).
*   **Hệ thống Workflow (Temporal):** Đây là "xương sống" của các tác vụ AI. HASH sử dụng Temporal để quản lý các luồng xử lý phức tạp, đảm bảo tính bền bỉ (reliability) và khả năng khôi phục khi các tác vụ AI dài hơi bị gián đoạn.
*   **Hệ thống định danh (Block Protocol):** Dự án tích hợp chặt chẽ với Block Protocol để tạo ra các giao diện và thực thể dữ liệu có cấu trúc mạnh (strongly-typed), cho phép dữ liệu và UI có thể tương tác lẫn nhau một cách nhất quán.
*   **Hạ tầng AI:** Tích hợp đa mô hình qua API của OpenAI, Anthropic (Claude), và Google Vertex AI.
*   **Cơ sở dữ liệu:** PostgreSQL (với Citus để mở rộng đồ thị), Redis (cho queue), và một lớp Graph truy vấn tùy chỉnh viết bằng Rust.

### 2. Tư duy Kiến trúc (Architectural Thinking)

HASH được kiến trúc theo mô hình **Self-building Knowledge Graph** (Đồ thị tri thức tự xây dựng):

*   **Kiến trúc Monorepo:** Sử dụng `Turborepo` và `Yarn Workspaces` để quản lý hàng chục ứng dụng và thư viện. Điều này giúp chia sẻ code (đặc biệt là các định nghĩa type) giữa Backend Rust và Frontend TS rất hiệu quả.
*   **Phân tách luồng xử lý (Execution Separation):**
    *   *Graph Layer:* Nơi lưu trữ thực thể và quan hệ.
    *   *API Layer:* Cổng giao tiếp GraphQL/REST.
    *   *Worker Layer:* Các tiến trình chạy ngầm xử lý việc "thu hoạch" dữ liệu từ web, phân tích văn bản và tự động điền vào đồ thị.
*   **Ontology-Driven:** Mọi thứ trong HASH đều dựa trên Schema. Hệ thống không chỉ lưu dữ liệu mà còn lưu cả định nghĩa của dữ liệu (Types). Khi AI tìm thấy thông tin mới, nó sẽ đối chiếu với Ontology hiện có để quyết định cách cấu trúc hóa thông tin đó.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Custom Query Language (HashQL):** Thay vì dùng SQL thuần, dự án xây dựng HashQL với bộ parser và compiler riêng trong Rust (`libs/@local/hashql`). Nó cho phép truy vấn đồ thị theo cách tối ưu hơn cho các thực thể AI.
*   **AI Agent Coordination:** Trong `apps/hash-ai-worker-ts/src/activities/flow-activities/research-entities-action/`, dự án triển khai các "Coordinating Agents". Các agent này có khả năng lập kế hoạch (plan), tự chia nhỏ nhiệm vụ, và gọi các sub-agents khác (như `link-follower-agent`) để đi tìm dữ liệu.
*   **Error-Stack & Diagnostics:** Sử dụng thư viện `error-stack` (trong Rust) để tạo ra các báo cáo lỗi có tính ngữ cảnh cao, cực kỳ quan trọng cho việc debug các hệ thống đồ thị phức tạp.
*   **Comptime & Codegen:** Sử dụng kỹ thuật sinh mã (codegen) tự động để đồng bộ hóa các kiểu dữ liệu từ Rust sang TypeScript, đảm bảo tính an toàn kiểu (type-safety) xuyên suốt hệ thống.

### 4. Luồng hoạt động hệ thống (System Workflows)

Một luồng tiêu biểu khi hệ thống tự động cập nhật dữ liệu:

1.  **Trigger:** Người dùng cung cấp một URL hoặc một câu hỏi (ví dụ: "Tìm các công ty con của Google").
2.  **Workflow Initiation:** Frontend gửi yêu cầu đến API, API khởi tạo một Temporal Workflow trong `hash-ai-worker-ts`.
3.  **Research Phase:**
    *   `web-search-action`: Gọi search engine để lấy danh sách các trang web liên quan.
    *   `get-web-page-activity`: Sử dụng Puppeteer để tải nội dung trang web.
    *   `sanitize-html`: Làm sạch HTML để tối ưu số lượng token cho LLM.
4.  **Inference Phase:**
    *   `infer-entities-from-content`: Gửi nội dung đã làm sạch tới LLM (GPT-4/Claude).
    *   LLM trích xuất các thực thể (entities) và mối quan hệ (claims).
5.  **Persistence Phase:**
    *   `persist-entities-action`: Hệ thống kiểm tra xem thực thể đã tồn tại chưa (deduplication).
    *   Dữ liệu mới được ghi vào `hash-graph` thông qua các transaction an toàn.
6.  **Real-time Update:** Dữ liệu mới xuất hiện trên UI của người dùng thông qua các thông báo và cập nhật đồ thị thời gian thực.

### Tổng kết
HASH là một hệ thống **Data-OS** hiện đại. Điểm mạnh nhất của nó là việc biến AI từ một công cụ "chat" thành một công cụ "cấu trúc hóa dữ liệu" thông qua việc kết hợp chặt chẽ giữa **Rust Performance**, **Temporal Workflows**, và **Strong Ontology**. Hệ thống này không chỉ lưu trữ thông tin mà còn chủ động đi tìm và làm giàu thông tin đó một cách tự động.