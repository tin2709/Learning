Dựa trên mã nguồn của dự án **Isartor**, đây là phân tích chi tiết về các khía cạnh kỹ thuật, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Isartor được xây dựng với tư duy "Pure Rust" nhằm đạt hiệu suất tối đa và khả năng triển khai linh hoạt (Single Binary).

*   **Ngôn ngữ & Runtime:** Rust (Edition 2024), chạy trên nền `tokio` (Async runtime).
*   **In-process ML (Candle):** Sử dụng framework `candle` (của HuggingFace) để chạy các mô hình AI trực tiếp trong process Rust mà không cần Python. Điều này cho phép thực hiện Semantic Cache (L1b) và SLM Router (L2) cực nhanh.
    *   Model nhúng: `all-MiniLM-L6-v2`.
    *   SLM nhúng: `Qwen-1.5B` (định dạng GGUF).
*   **Web Framework & Networking:**
    *   `Axum`: Dùng cho API Gateway.
    *   `Tower`: Xây dựng hệ thống middleware theo tầng.
    *   `tokio-rustls`: Xử lý TLS và MITM proxy.
*   **Storage & Caching:** `ahash` cho hashing tốc độ cao, `lru` cho cache bộ nhớ trong, và hỗ trợ `Redis` cho môi trường Cluster.
*   **LLM Abstraction:** Sử dụng `rig-core` để thống nhất giao tiếp với nhiều Cloud Provider (OpenAI, Anthropic, Azure, Groq...).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Isartor xoay quanh khái niệm **"Deflection Stack" (Chồng lớp chuyển hướng)**. Mục tiêu là ngăn chặn traffic prompt dư thừa rời khỏi hạ tầng của người dùng.

*   **Kiến trúc phân tầng (Cascading Layers):** Hệ thống được thiết kế như một bộ lọc nhiều lớp. Càng ở tầng thấp (L1a, L1b), độ trễ càng nhỏ (< 1ms) và chi phí càng rẻ ($0).
*   **Local-first & Privacy-centric:** Ưu tiên xử lý tại chỗ. Isartor có thể chạy trong môi trường Air-gapped (ngắt kết nối internet hoàn toàn) nếu sử dụng các model local.
*   **Agnostic Interface:** Hỗ trợ đa dạng bề mặt client (OpenAI-compatible, Anthropic-compatible, MCP, và CONNECT proxy). Isartor đóng vai trò như một "Universal Adaptor".
*   **Middleware-Driven:** Toàn bộ logic kiểm soát (auth, monitoring, cache, triage) được đóng gói thành các middleware của Tower, cho phép mở rộng hoặc thay đổi thứ tự lọc một cách linh hoạt.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Body Buffering (BufferedBody):** Vì các layer middleware cần đọc request body nhiều lần (để hash, để phân tích ngữ nghĩa), Isartor sử dụng kỹ thuật sao chép body vào request extensions (`src/middleware/body_buffer.rs`). Đây là kỹ thuật quan trọng trong Axum để tránh lỗi "body consumed".
*   **Trait-based Extensibility:**
    *   `CompressionStage`: Cho phép thêm các thuật toán nén context mới vào L2.5.
    *   `AppLlmAgent`: Trừu tượng hóa các provider LLM ở L3.
*   **Staged Pipeline Pattern:** Đặc biệt trong L2.5 (Context Optimizer), dữ liệu đi qua một pipeline gồm: `ContentClassifier` -> `DedupStage` -> `LogCrunchStage`. Mỗi stage tập trung vào một nhiệm vụ nén/tối ưu cụ thể.
*   **Deterministic Hashing:** Sử dụng SHA-256 trên một cấu trúc prompt ổn định (stable prompt extraction) để làm khóa cho L1a, đảm bảo các prompt giống hệt nhau luôn có cùng một key.
*   **Semantic Similarity:** Sử dụng Cosine Similarity tính toán bằng vector embeddings để phát hiện các câu hỏi có ý nghĩa tương đương (ví dụ: "Giá bao nhiêu?" và "Sản phẩm này tốn bao nhiêu tiền?").

### 4. Luồng hoạt động hệ thống (System Operational Flow)

Một request đi qua Isartor sẽ trải qua luồng xử lý sau:

1.  **Nhận Request:** Axum router nhận request từ nhiều nguồn (API `/v1/chat`, CONNECT proxy, hoặc MCP).
2.  **Định danh & Buffer:** Middleware định danh tool AI đang sử dụng (qua User-Agent) và lưu body vào bộ đệm.
3.  **Lớp L1a (Exact Cache):** Hash toàn bộ prompt. Nếu khớp khóa trong bộ nhớ/Redis, trả kết quả ngay lập tức (latency < 1ms).
4.  **Lớp L1b (Semantic Cache):** Nếu L1a miss, chuyển prompt qua model nhúng để lấy vector. So sánh độ tương đồng với các entry cũ. Nếu similarity > threshold (0.85), trả kết quả từ cache (latency 1-5ms).
5.  **Lớp L2 (SLM Router):** Nếu cả 2 cache đều miss, một mô hình ngôn ngữ nhỏ (SLM) local sẽ phân tích xem câu hỏi có đơn giản không. Nếu đơn giản (FAQ, greeting...), SLM tự trả lời luôn.
6.  **Lớp L2.5 (Context Optimizer):** Nếu phải đi ra Cloud, middleware này sẽ kiểm tra và nén các instruction dài (như `CLAUDE.md`) bằng cách xóa comment HTML, rút gọn khoảng trắng hoặc dùng mã băm phiên làm việc.
7.  **Lớp L3 (Cloud Logic):** Gửi prompt đã tối ưu đến Cloud Provider. Kết quả trả về sẽ được lưu ngược lại vào các lớp Cache để phục vụ các request sau.
8.  **Trả kết quả:** Trả response kèm các header quan trọng như `X-Isartor-Layer` và `X-Isartor-Deflected` để người dùng biết request đã được xử lý ở tầng nào.

### Tổng kết
Isartor không chỉ là một Gateway đơn thuần; nó là một **"Prompt Firewall"** thông minh. Điểm mạnh nhất của nó là việc tích hợp **In-process Inference bằng Rust**, giúp nó thực hiện được các tác vụ AI phức tạp ngay trên perimeter mà không cần phụ thuộc vào một stack Python cồng kềnh.