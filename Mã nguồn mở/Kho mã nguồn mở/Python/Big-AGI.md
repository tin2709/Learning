Dựa trên tài liệu và cấu trúc mã nguồn của dự án **Big-AGI**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật:

### 1. Công nghệ cốt lõi (Core Technologies)
Big-AGI là một ứng dụng Web hiện đại, được tối ưu hóa cực độ cho hiệu suất và khả năng mở rộng:

*   **Frontend Framework:** Next.js 15 (sử dụng cả App Router cho API và Pages Router cho UI), React 18.
*   **Ngôn ngữ:** TypeScript (chiếm >80%), đảm bảo tính an toàn về kiểu dữ liệu (type-safe) trên toàn hệ thống.
*   **Giao diện & Styling:** Material-UI (Joy UI) cho thiết kế chuyên nghiệp, kết hợp với Emotion (CSS-in-JS).
*   **Quản lý trạng thái (State Management):** Sử dụng **Zustand**. Đây là lựa chọn then chốt vì nó nhẹ và hiệu năng cao, cho phép lưu trữ trạng thái vào `localStorage` hoặc `IndexedDB` để hỗ trợ kiến trúc "local-first".
*   **API & Giao tiếp:** **tRPC** cùng với **TanStack React Query**, tạo ra luồng dữ liệu type-safe từ Backend tới Frontend.
*   **Runtime Environments:**
    *   **Edge Runtime:** Dùng cho các tác vụ AI cần độ trễ thấp (streaming).
    *   **Node.js:** Dùng cho các tác vụ xử lý dữ liệu nặng như duyệt web (Browsing) hoặc xuất file (Trade).
*   **Database:** Prisma ORM, hỗ trợ Postgres (cho môi trường cloud/serverless) và MongoDB.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Hệ thống được thiết kế theo hướng **Modular & Local-first**:

*   **Cấu trúc "Apps" độc lập:** Các tính năng lớn (`chat`, `call`, `beam`, `draw`) được đóng gói trong thư mục `src/apps/`. Mỗi app có UI và trạng thái (store) riêng, giúp dễ dàng bảo trì và mở rộng.
*   **Kiến trúc AIX (AI Exchange):** Đây là framework trung tâm để giao tiếp với AI. Nó trừu tượng hóa các giao thức khác nhau (OpenAI, Anthropic, Gemini) thành một định dạng chung (Particle-based streaming), cho phép hỗ trợ hơn 20 nhà cung cấp chỉ qua một adapter duy nhất.
*   **Phân tách Server (Edge vs Cloud):** 
    *   `trpc.router-edge`: Xử lý các tác vụ AI nhanh.
    *   `trpc.router-cloud`: Xử lý các tác vụ logic phức tạp.
*   **Kiến trúc Local-first:** Big-AGI ưu tiên xử lý và lưu trữ dữ liệu tại trình duyệt của người dùng. Backend chủ yếu đóng vai trò là proxy chuyển tiếp yêu cầu đến các API AI.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)
Mã nguồn Big-AGI thể hiện trình độ xử lý TypeScript và React rất cao:

*   **Particle-based Streaming:** Kỹ thuật chia nhỏ phản hồi từ AI thành các "hạt" dữ liệu, giúp UI cập nhật mượt mà theo thời gian thực mà không cần chờ đợi toàn bộ câu trả lời.
*   **CSF (Client-Side Fetch):** Cho phép trình duyệt gọi trực tiếp API của các nhà cung cấp (như Ollama, LM Studio) nếu hỗ trợ CORS, bỏ qua Server của Big-AGI để giảm độ trễ tối đa.
*   **Mô hình DMessage & Fragments:** Thay vì lưu tin nhắn là một chuỗi văn bản đơn giản, Big-AGI lưu dưới dạng một mảng các `fragments` (văn bản, code, hình ảnh, công cụ). Điều này giúp hệ thống render cực kỳ linh hoạt (Markdown, LaTeX, Mermaid, SVG).
*   **Zustand Persistence & Migrations:** Sử dụng middleware để tự động lưu trạng thái vào IndexedDB. Hệ thống có cơ chế `migrations` để tự động nâng cấp cấu trúc dữ liệu khi ứng dụng cập nhật phiên bản mới mà không làm mất dữ liệu người dùng.
*   **Conditional Module Replacement:** Sử dụng Webpack để thay thế các module server-only bằng các bản "mock" ở phía client, giúp tái sử dụng mã nguồn mà không làm gãy quá trình build bundle.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Chuẩn bị (Bootstrap):** Khi mở app, `ProviderBootstrapLogic` sẽ kiểm tra cấu hình, preload thư viện tokenizer (Tiktoken) và tải dữ liệu từ IndexedDB vào Zustand store.
2.  **Tiếp nhận Input:** Người dùng nhập liệu qua `Composer`. Tin nhắn được chuyển đổi thành cấu trúc `DMessage`.
3.  **Điều phối AI (AIX Client):** Yêu cầu được gửi qua AIX Client. Tùy cấu hình, nó sẽ quyết định đi qua Server (tRPC) hoặc gọi trực tiếp (CSF).
4.  **Xử lý tại Backend (Adapter):** Adapter tương ứng (ví dụ: `anthropic.messageCreate.ts`) sẽ chuyển đổi bối cảnh tin nhắn sang giao thức của nhà cung cấp AI.
5.  **Luồng phản hồi (Streaming):** AI phản hồi -> Server thu nhận stream -> AIX Transmitter gửi các "particles" về Client -> `ContentReassembler` lắp ghép lại thành Fragments.
6.  **Hiển thị (Rendering):** UI lắng nghe thay đổi trong Zustand store và render kết quả qua các Block chuyên dụng (Markdown block, Code block với tính năng highlight).
7.  **Lưu trữ (Persistence):** Ngay khi có dữ liệu mới, Zustand tự động đồng bộ xuống IndexedDB trên máy người dùng.

### Tổng kết
Big-AGI là một nền tảng AI chuyên sâu, không chỉ là một giao diện chat đơn giản mà là một **Hệ sinh thái làm việc với AI**. Nó giải quyết bài toán "Vendor Lock-in" (phụ thuộc một nhà cung cấp) bằng kiến trúc Adapter linh hoạt và bảo vệ quyền riêng tư người dùng thông qua mô hình lưu trữ cục bộ mạnh mẽ.