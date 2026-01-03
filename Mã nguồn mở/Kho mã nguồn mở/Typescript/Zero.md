Dưới đây là bản phân tích chi tiết về dự án **Zero (Mail-0)** - một giải pháp thay thế Gmail mã nguồn mở tích hợp AI, được trình bày theo cấu trúc chuyên sâu để đưa vào tệp README.

---

# 🛡️ Phân tích Kỹ thuật Dự án Zero (Mail-0)

## 1. Công nghệ cốt lõi (Core Tech Stack)
Dự án được xây dựng trên một hệ sinh thái hiện đại, tối ưu cho hiệu suất cực cao và khả năng mở rộng:

*   **Frontend:**
    *   **React 19 & Next.js (React Router v7):** Sử dụng các tính năng mới nhất của React để tối ưu hóa việc render.
    *   **Tailwind CSS v4:** Tận dụng kiến trúc engine mới "Oxide" cho tốc độ biên dịch nhanh và cấu hình trực tiếp trong CSS.
    *   **Jotai:** Quản lý state nguyên tử (atomic state), giúp kiểm soát trạng thái UI phức tạp mà không gây re-render dư thừa.
    *   **Novel & Tiptap:** Trình soạn thảo văn bản giàu tính năng, hỗ trợ các lệnh slash (/) và autocomplete bằng AI.
*   **Backend & Infrastructure:**
    *   **Cloudflare Workers & Hono:** Backend chạy trên Edge Computing, đảm bảo độ trễ thấp nhất toàn cầu.
    *   **tRPC:** Đảm bảo Type-safe tuyệt đối giữa Client và Server.
    *   **Durable Objects (DO):** Lưu trữ trạng thái và xử lý logic riêng biệt cho từng người dùng, hỗ trợ tính năng real-time (WebSockets).
    *   **Cloudflare Workflows:** Xử lý các tác vụ nền phức tạp như đồng bộ hóa hàng triệu email theo chu kỳ.
*   **Database & Storage:**
    *   **PostgreSQL & Drizzle ORM:** Cơ sở dữ liệu chính với Hyperdrive để tối ưu kết nối từ Edge.
    *   **Cloudflare R2:** Lưu trữ tệp đính kèm và nội dung email thô (raw content).
    *   **Cloudflare Vectorize:** Cơ sở dữ liệu vector phục vụ tính năng tìm kiếm thông minh (RAG).
*   **AI Integration:**
    *   **Vercel AI SDK:** Hỗ trợ đa mô hình (OpenAI, Anthropic, Google Gemini, Groq, Perplexity).
    *   **ElevenLabs:** Tích hợp trợ lý giọng nói để tương tác với hộp thư qua cuộc gọi.

## 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Zero tập trung vào ba trụ cột: **Quyền riêng tư, Tốc độ và Sự thông minh.**

*   **Kiến trúc Monorepo:** Sử dụng `pnpm workspaces` và `Turbo` để quản lý đồng thời ứng dụng Mail, Server và các gói dùng chung (DB, Cli, Testing), giúp tăng tốc độ phát triển và đồng bộ hóa logic.
*   **DO Sharding Logic:** Thay vì lưu trữ tập trung, dữ liệu người dùng được phân mảnh (sharding) qua các Durable Objects. Điều này giúp tránh nghẽn cổ chai và cho phép mỗi người dùng có một "máy chủ nhỏ" riêng để xử lý AI.
*   **Edge-First Design:** Hầu hết logic xử lý email và AI được thực hiện tại Edge, giúp giảm tải cho database trung tâm và tăng tốc độ phản hồi UI.
*   **URL-as-State:** Sử dụng `nuqs` để quản lý trạng thái ứng dụng (như `threadId`, `folder`, `mode`) trực tiếp qua URL, giúp người dùng dễ dàng chia sẻ trạng thái hoặc quay lại trang trước đó.

## 3. Các kỹ thuật chính (Key Techniques)
Dự án áp dụng nhiều kỹ thuật phần mềm nâng cao:

*   **Optimistic Updates (Cập nhật lạc quan):** Sử dụng `optimistic-actions-manager` kết hợp với Jotai để UI phản hồi ngay lập tức (Xóa, Lưu trữ, Gắn sao) trước khi server phản hồi, mang lại cảm giác cực kỳ mượt mà.
*   **Writing Style Mirroring:** Sử dụng thuật toán **Welford Variance** để phân tích thói quen viết lách của người dùng (độ dài câu, cách dùng từ, emoji, thái độ). Từ đó, AI có thể soạn thảo email giả lập phong cách riêng của từng người.
*   **RAG (Retrieval-Augmented Generation):** Email được chuyển đổi thành vector embedding và lưu vào `Vectorize`. Khi người dùng hỏi "Tìm hóa đơn tháng trước", hệ thống sẽ tìm kiếm theo ngữ nghĩa thay vì chỉ tìm từ khóa.
*   **Email Verification Engine:** Tích hợp bộ kiểm tra SPF, DKIM, DMARC và BIMI thủ công để xác thực danh tính người gửi, bảo vệ người dùng khỏi email giả mạo.
*   **Image Compression:** Tự động nén hình ảnh đính kèm ngay tại client trước khi gửi để tiết kiệm băng thông và dung lượng.

## 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Đăng nhập & Kết nối:** Người dùng đăng nhập qua Google/Microsoft OAuth (Better Auth). Hệ thống tạo một ConnectionID và khởi tạo không gian lưu trữ riêng trong Durable Object.
2.  **Đồng bộ hóa (Sync):** 
    *   Cloudflare Workflows được kích hoạt để quét hộp thư.
    *   Nội dung email được lưu vào R2, metadata được lưu vào SQLite (trong DO).
    *   AI tiến hành tóm tắt và đánh dấu nhãn tự động cho các email quan trọng.
3.  **Xử lý AI (Agentic Workflow):** 
    *   Khi người dùng yêu cầu (qua Chat hoặc Voice), AI Agent sẽ xác định ý định.
    *   Sử dụng **Tool Calling** để thực hiện các tác vụ như: `listThreads`, `createDraft`, `modifyLabels`, hoặc `webSearch` (qua Perplexity).
4.  **Tương tác & Phản hồi:** 
    *   Người dùng soạn thảo thư bằng Novel Editor với sự hỗ trợ của AI Autocomplete.
    *   Các hành động như "Lưu trữ" hay "Xóa" được thực hiện qua hàng đợi (Background Queue) để đảm bảo tính nhất quán kể cả khi mạng yếu.
5.  **Gửi thư & Hoàn tác:** Email được chuyển đổi sang định dạng MIME -> Gửi qua API của nhà cung cấp. Người dùng có 15-30 giây để nhấn "Undo Send" nhờ cơ chế trì hoãn (Delayed Queue) trong KV.

---
*Zero không chỉ là một ứng dụng mail, nó là một hệ điều hành thông minh cho giao tiếp điện tử.*