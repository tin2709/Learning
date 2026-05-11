Dựa trên các tệp tin và cấu trúc thư mục của dự án **Leon AI** (phiên bản 2.0 Developer Preview), dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và kỹ thuật của hệ thống trợ lý ảo cá nhân mã nguồn mở này:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Dự án sử dụng một hệ sinh thái đa ngôn ngữ và hiện đại, tập trung vào hiệu suất và khả năng mở rộng:

*   **Runtime chính:** **Node.js v24+** kết hợp với **TypeScript**. Việc sử dụng phiên bản Node.js mới nhất cho thấy dự án tận dụng các tính năng ES Modules và hiệu suất xử lý bất đồng bộ hiện đại.
*   **Web Framework & Real-time:** 
    *   **Fastify:** Được dùng làm server HTTP chính nhờ tốc độ vượt trội so với Express.
    *   **Socket.io:** Đảm bảo giao tiếp hai chiều thời gian thực giữa client và server (stream token LLM, trạng thái xử lý).
*   **Hệ thống AI & LLM:**
    *   **Vercel AI SDK:** Thư viện nòng cốt để tích hợp đa dạng các nhà cung cấp LLM (OpenAI, Anthropic, Groq...).
    *   **Local LLM:** Hỗ trợ chạy mô hình cục bộ qua `node-llama-cpp` và `sglang`, nhấn mạnh tính riêng tư (Privacy-first).
*   **Frontend:** 
    *   **React** kết hợp với **Vite** để build ứng dụng web nhanh chóng.
    *   **Aurora:** Một bộ thư viện UI (Design System) riêng của Leon được xây dựng bằng TypeScript và Sass, cho phép các "Skill" hiển thị các widget tương tác ngay trong khung chat.
*   **Ngôn ngữ bổ trợ:** **Python** được sử dụng cho các service xử lý tín hiệu âm thanh (STT/TTS) và cho phép viết Skill bằng Python thông qua một hệ thống "Bridge".

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Leon 2.0 đã chuyển mình từ "intent-classification" đơn giản sang **"Agentic Architecture"**:

*   **Phân lớp thực thi (Hierarchical Execution):** Leon tổ chức khả năng theo thứ tự: `Skills -> Actions -> Tools -> Functions -> Binaries`. Cách tiếp cận này giúp cô lập logic nghiệp vụ khỏi các công cụ thực thi cấp thấp.
*   **Chế độ thực thi lai (Hybrid Modes):**
    *   `Smart`: Tự động quyết định luồng đi.
    *   `Controlled`: Chạy các skill định nghĩa sẵn theo quy trình cứng (Deterministic), đảm bảo độ tin cậy.
    *   `Agent`: Lập kế hoạch từng bước (Planning) để giải quyết các yêu cầu phức tạp.
*   **Kiến trúc Bridge:** Điểm đặc biệt của Leon là khả năng chạy code đa ngôn ngữ. Core hệ thống bằng Node.js có thể điều phối các Skill viết bằng Python thông qua một giao thức giao tiếp chuẩn hóa, giúp tận dụng thế mạnh của thư viện Python trong khoa học dữ liệu/AI.
*   **Layered Memory (Bộ nhớ phân lớp):** Leon không chỉ lưu log. Nó chia bộ nhớ thành: Durable preferences (sở thích lâu dài), Day-to-day context (ngữ cảnh hàng ngày) và Recent context (hội thoại gần đây).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Aurora Component Rendering:** Leon có một kỹ thuật cho phép "Brain" gửi định nghĩa component Tree từ server xuống client qua Socket.io. Client sau đó sử dụng `renderAuroraComponent.js` để dựng lại giao diện React động, giúp các kỹ năng (Skills) có giao diện tương tác phong phú mà không cần nạp lại trang.
*   **Context Files (.md):** Leon sử dụng các file Markdown (`LEON.md`, `ARCHITECTURE.md`) làm "grounding data" cho chính nó. Hệ thống tự quét các file này để hiểu về bản thân và cấu trúc hệ thống, một kỹ thuật giúp mô hình ngôn ngữ bám sát thực tế (grounded behavior).
*   **Managed Node/Python Runtimes:** Thư mục `scripts/setup/` chứa các đoạn mã tự động quản lý môi trường (managed runtimes), tự động cài đặt `uv` (cho Python) và `pnpm` (cho Node.js), giảm thiểu rắc rối khi thiết lập cho người dùng cuối.
*   **TCP Server cho Audio:** Leon sử dụng một server TCP bằng Python (`tcp_server/`) chuyên dụng để xử lý luồng âm thanh STT/TTS, tách biệt gánh nặng tính toán xử lý tín hiệu ra khỏi main loop của Node.js.

### 4. Luồng Hoạt động Hệ thống (System Operation Flow)

1.  **Tiếp nhận (Input):** User gửi tin nhắn văn bản hoặc giọng nói (qua STT Bridge). client-side sử dụng `Recorder.js` và `VoiceEnergy.js` để trực quan hóa năng lượng giọng nói.
2.  **Xử lý ngôn ngữ (NLU/LLM Manager):** 
    *   Input đi vào `Brain`. Nếu ở chế độ `Agent`, `Planning LLM Duty` sẽ tạo ra một bản kế hoạch.
    *   Hệ thống quyết định cần gọi `Tool` nào (ví dụ: `tiny-web-crawler`).
3.  **Thực thi (Execution Bridge):** Server gửi yêu cầu đến Bridge tương ứng (Node.js hoặc Python). Tool thực thi nhiệm vụ (ví dụ: lấy dữ liệu từ web) và trả về JSON.
4.  **Phản hồi (NLG & UI):** 
    *   Dữ liệu thô được LLM chuyển thành văn bản tự nhiên.
    *   Nếu Skill có widget, server gửi `componentTree` qua Socket.io.
    *   Client render widget và hiển thị token văn bản theo kiểu "streaming" để tạo cảm giác phản hồi tức thì.
5.  **Hậu xử lý:** Hệ thống cập nhật `Memory` và `Context` để phục vụ các yêu cầu tiếp theo.

### Tổng kết
**Leon AI** không còn là một chatbot đơn thuần mà là một khung làm việc (framework) cho **AI Agent**. Sự kết hợp giữa khả năng thực thi deterministic (quy trình cứng) và stochastic (LLM linh hoạt) cùng với hệ thống UI Widget động khiến Leon trở thành một trong những dự án trợ lý ảo mã nguồn mở có tư duy kiến trúc bài bản và thực dụng nhất hiện nay.