Dưới đây là phân tích chi tiết về dự án **Cossistant**, một nền tảng hỗ trợ khách hàng mã nguồn mở dựa trên AI, tối ưu cho hệ sinh thái React/Next.js.

---

### 1. Công nghệ cốt lõi (Core Technology)

Cossistant được xây dựng trên một "modern stack" cực kỳ tối ưu cho hiệu suất và trải nghiệm nhà phát triển:

*   **Runtime & Package Manager:** **Bun** là lựa chọn chủ đạo, được dùng làm runtime cho cả API và Worker, giúp tăng tốc độ thực thi và quản lý monorepo qua **Turborepo**.
*   **Backend API:** Sử dụng **Hono** (một web framework siêu nhanh) kết hợp với **tRPC**. Sự kết hợp này mang lại khả năng "End-to-end Type Safety" (an toàn kiểu dữ liệu từ DB đến tận UI).
*   **AI Orchestration:** Sử dụng **Vercel AI SDK** kết hợp với **OpenRouter** để linh hoạt chuyển đổi giữa các mô hình LLM (như GPT-4, Claude). Dự án tích hợp **pgvector** trong PostgreSQL để xử lý RAG (Retrieval-Augmented Generation).
*   **Real-time Analytics:** **Tinybird** được dùng làm nền tảng xử lý dữ liệu thời gian thực. Nó xử lý hàng triệu sự kiện (presence, page views) để dựng lên bản đồ "Live Visitor Globe" và các chỉ số inbox analytics mà không làm chậm DB chính.
*   **Infrastructure:** PostgreSQL (DB chính), Redis (Caching & Queuing), và Docker cho việc đóng gói môi trường.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án áp dụng triết lý **"Headless-first & Decoupled"**:

*   **Cấu trúc Monorepo:** Tách biệt rõ ràng các logic nghiệp vụ thành các package riêng biệt (`@cossistant/core`, `@cossistant/react`, `@cossistant/types`), cho phép tái sử dụng code giữa Web app, API và Workers.
*   **Kiến trúc Pipeline AI phân rã:** Đây là điểm sáng nhất. AI không chỉ phản hồi tin nhắn; nó hoạt động qua 2 luồng:
    1.  **Primary Pipeline:** Xử lý thời gian thực, quyết định AI có nên trả lời ngay không (Decision) và thực hiện các lời gọi công cụ (Tool calls).
    2.  **Background Pipeline:** Chạy ngầm (delay 30s), thực hiện các tác vụ "triage" như phân tích cảm xúc (sentiment), tóm tắt tiêu đề và phân loại hội thoại để giảm tải cho con người.
*   **Hybrid Deployment:** Kiến trúc hỗ trợ cả mô hình Cloud (SaaS) và Self-hosted thông qua Terraform scripts cho AWS (S3, SES), thể hiện tư duy linh hoạt về hạ tầng.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Type-safe Database Schema:** Sử dụng **Drizzle ORM**. Kỹ thuật này giúp schema của database trở thành "nguồn sự thật duy nhất" (Single source of truth) cho toàn bộ ứng dụng TypeScript.
*   **Custom WebSocket Registry:** Thay vì dùng các thư viện nặng, dự án tự triển khai `connection-registry.ts` để quản lý các kết nối WebSocket, đảm bảo việc đẩy thông báo (seen, typing, new message) đến đúng đối tượng (visitor hoặc team member).
*   **Telemetry & Redaction:** Kỹ thuật sanitize dữ liệu trong luồng AI pipeline để đảm bảo thông tin nhạy cảm không bị lộ trong log hoặc gửi đi các bên thứ ba (LLM providers).
*   **Facehash Generation:** Một package riêng (`facehash`) dùng thuật toán để tạo ra các avatar SVG duy nhất dựa trên ID người dùng, giúp cá nhân hóa khách truy cập mà không cần ảnh thật.

---

### 4. Luồng hoạt động hệ thống (System Flow)

1.  **Giai đoạn Tương tác (Visitor Interaction):**
    *   Khách truy cập vào website -> Widget `<Support />` kích hoạt -> Gửi sự kiện `presence` đến Tinybird (qua scoped JWT).
    *   Khách gửi tin nhắn -> Hono API tiếp nhận -> Lưu vào Postgres -> Đẩy job vào BullMQ (Redis).
2.  **Giai đoạn Xử lý AI (AI Processing):**
    *   **Worker** lấy job -> Chạy **Primary Pipeline**.
    *   **Intake:** Nạp ngữ cảnh (lịch sử chat, kiến thức từ RAG).
    *   **Decision:** AI dùng "Smart Decision Policy" để xác định: "Tôi có đủ kiến thức trả lời không? Khách có đang giận dữ cần người thật không?".
    *   **Generation:** Nếu trả lời, AI thực hiện Tool call (như `sendMessage`) hoặc yêu cầu làm rõ kiến thức (Clarification).
3.  **Giai đoạn Hậu xử lý (Background Triage):**
    *   Sau khi Primary hoàn tất, một job **Background Pipeline** được lên lịch sau 30 giây.
    *   Hệ thống tự động cập nhật Metadata: "Sentiment: Negative", "Priority: High", "Title: Vấn đề thanh toán".
4.  **Giai đoạn Phản hồi thời gian thực (Live Dashboard):**
    *   Mọi thay đổi từ AI được phát qua **Realtime Emitter**.
    *   Dashboard của nhân viên hỗ trợ cập nhật ngay lập tức nhờ tRPC Subscription/WebSocket mà không cần reload trang.

### Tổng kết
Cossistant không chỉ là một widget chat đơn thuần; nó là một **hệ điều hành hỗ trợ khách hàng** thu nhỏ. Sự kết hợp giữa hạ tầng xử lý dữ liệu mạnh mẽ (Tinybird) và logic AI phân tầng (Primary/Background) biến nó thành một giải pháp chuyên nghiệp cho các startup SaaS muốn tối ưu hóa hỗ trợ khách hàng bằng AI.