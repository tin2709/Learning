Dựa trên mã nguồn và tài liệu của dự án Vanna 2.0, dưới đây là phân tích chi tiết về các khía cạnh công nghệ và kiến trúc:

### 1. Công nghệ cốt lõi (Core Technology)

Vanna 2.0 là một hệ thống **Agentic AI** chuyên sâu cho việc chuyển đổi ngôn ngữ tự nhiên thành SQL và phân tích dữ liệu.
*   **Ngôn ngữ & Framework:** Python 3.9+ (Backend), TypeScript với Lit (Frontend Web Components).
*   **LLM Agnostic (Đa mô hình):** Tích hợp sâu với các nhà cung cấp lớn như OpenAI, Anthropic (Claude), Google Gemini, và các mô hình chạy local qua Ollama thông qua các lớp trừu tượng `LlmService`.
*   **Vector Databases:** Hỗ trợ đa dạng các cơ sở dữ liệu vector (ChromaDB, Qdrant, FAISS, Milvus, Pinecone) để lưu trữ "Memory" (các câu lệnh SQL đúng mẫu, DDL, tài liệu).
*   **Pydantic V2:** Sử dụng Pydantic làm xương sống cho việc định nghĩa dữ liệu (Models), kiểm tra kiểu (Type Checking) và tự động tạo JSON Schema cho các Tools để LLM dễ dàng sử dụng (Function Calling).
*   **Streaming & Real-time:** Sử dụng Server-Sent Events (SSE) và WebSockets để truyền tải phản hồi theo thời gian thực (Progress updates, streaming text, interactive tables).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Vanna 2.0 chuyển dịch từ một thư viện đơn giản sang một **Framework Agent hướng doanh nghiệp**.
*   **User-Aware Scoping (Nhận diện người dùng):** Đây là tư duy quan trọng nhất. Danh tính người dùng (`User`) không chỉ nằm ở lớp Auth mà chảy xuyên suốt qua mọi lớp: từ `System Prompt`, việc thực thi `Tool`, đến việc lọc kết quả SQL.
*   **Modular Tool System (Hệ thống công cụ mô-đun):** Thay vì viết code xử lý tập trung, Vanna chia nhỏ các khả năng (chạy SQL, vẽ biểu đồ, gửi email) thành các `Tool`. Mỗi Tool có định nghĩa đầu vào (Arguments) và quyền truy cập (`access_groups`) riêng.
*   **Separation of Concerns (Phân rã trách nhiệm):**
    *   `UserResolver`: Định nghĩa cách lấy danh tính từ Request (JWT, Cookies).
    *   `ToolRegistry`: Quản lý danh sách và kiểm soát quyền thực thi các công cụ.
    *   `SqlRunner`: Trừu tượng hóa việc kết nối và thực thi trên các DB khác nhau (Postgres, Snowflake, BigQuery...).
*   **Component-Driven UI (Giao diện hướng thành phần):** Backend không chỉ gửi văn bản thô mà gửi các `RichComponent` (Table, Chart, StatusCard). Frontend (`vanna-chat`) chỉ việc render các thành phần này theo trạng thái nhận được từ luồng streaming.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Dependency Injection (Tiêm phụ thuộc):** Hầu hết các service (`SqlRunner`, `LlmService`, `AgentMemory`) được tiêm vào Agent thông qua constructor, giúp việc kiểm thử (unit test) và chuyển đổi môi trường cực kỳ linh hoạt.
*   **Middleware & Lifecycle Hooks:** Sử dụng các lớp trừu tượng như `LlmMiddleware` và `LifecycleHook`. Kỹ thuật này cho phép can thiệp vào trước/sau mỗi lời gọi LLM hoặc trước/sau mỗi tin nhắn để xử lý cache, kiểm tra hạn ngạch (quota) hoặc ghi log audit.
*   **Asynchronous Generators:** Sử dụng `async for` và `yield` rộng rãi để xử lý luồng dữ liệu streaming, giúp giao diện người dùng cực kỳ mượt mà (nhìn thấy Agent đang "suy nghĩ" hoặc "chạy tool" ngay lập tức).
*   **Argument Transformation (RLS):** Kỹ thuật `transform_args` trong `ToolRegistry` cho phép backend tự động chèn thêm các điều kiện lọc dữ liệu (Row-level security) vào câu lệnh SQL mà người dùng (hoặc LLM) không thể can thiệp trực tiếp từ prompt.
*   **Adapter Pattern:** `LegacyVannaAdapter` là một ví dụ điển hình của việc giúp người dùng phiên bản cũ (0.x) chuyển dịch sang 2.0 mà không cần viết lại toàn bộ logic cũ.

### 4. Luồng hoạt động hệ thống (System Operational Flow)

1.  **Resolve User:** Khi có request từ `<vanna-chat>`, `UserResolver` trích xuất thông tin người dùng từ request context.
2.  **Workflow Pre-processing:** `WorkflowHandler` kiểm tra xem tin nhắn có phải là lệnh đặc biệt (như `/help`, `/status`) hay không. Nếu có, nó xử lý và trả về kết quả ngay mà không gọi LLM.
3.  **Context Enhancement:** `LlmContextEnhancer` tìm kiếm trong `AgentMemory` các mẫu SQL hoặc tài liệu liên quan đến câu hỏi để chèn vào prompt, giúp LLM hiểu ngữ cảnh dữ liệu của khách hàng (RAG).
4.  **Agent Reasoning Loop:** 
    *   LLM nhận prompt và quyết định gọi Tool (ví dụ: `run_sql`).
    *   Agent thực thi Tool thông qua `ToolRegistry` (có kiểm tra quyền).
    *   Kết quả Tool được gửi ngược lại cho LLM để tóm tắt hoặc gọi thêm Tool khác.
5.  **Streaming UI Components:** Trong suốt quá trình loop, các `ChatStreamChunk` chứa `RichComponent` được gửi về frontend qua SSE. Người dùng thấy bảng dữ liệu hiện ra trước, sau đó là biểu đồ Plotly, và cuối cùng là tóm tắt bằng ngôn ngữ tự nhiên.
6.  **Audit & Observability:** Mọi hành động được `AuditLogger` ghi lại và `ObservabilityProvider` theo dõi hiệu năng (tokens, thời gian phản hồi).

### Tổng kết
Vanna 2.0 không chỉ là một công cụ Text-to-SQL, nó là một giải pháp **Enterprise Data Agent** hoàn chỉnh, giải quyết các bài toán hóc búa về bảo mật dữ liệu, quản lý quyền và trải nghiệm người dùng trong môi trường doanh nghiệp thực tế.