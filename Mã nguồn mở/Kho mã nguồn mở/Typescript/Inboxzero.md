Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Inbox Zero**:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án được xây dựng trên một Stack hiện đại, tối ưu cho tốc độ phát triển và khả năng mở rộng:
*   **Framework:** **Next.js 15+ (App Router)** - Sử dụng React 19, tối ưu hóa phía Server (SSR) và Client.
*   **Ngôn ngữ:** **TypeScript** (chiếm 99.1%) đảm bảo tính chặt chẽ về kiểu dữ liệu.
*   **Cơ sở dữ liệu:** **PostgreSQL** với **Prisma ORM** để quản lý Schema và Migration.
*   **Caching & Queuing:** **Redis (Upstash)** dùng cho Rate limiting, lưu trữ session và hàng đợi xử lý.
*   **AI SDK:** Sử dụng **Vercel AI SDK**, hỗ trợ đa mô hình (OpenAI, Anthropic, Google Gemini, Groq, Ollama).
*   **Giao diện:** **Tailwind CSS** kết hợp với **shadcn/ui**, **Framer Motion** cho hiệu ứng mượt mà.
*   **Quản lý Monorepo:** **Turborepo** và **pnpm** giúp quản lý nhiều package/app trong cùng một kho lưu trữ.
*   **Xử lý Email:** Kết nối qua **Google APIs (Gmail)** và **Microsoft Graph API (Outlook)**.

### 2. Kỹ thuật và Tư duy kiến trúc (Architectural Thinking)
*   **Cấu trúc Monorepo:** Chia nhỏ hệ thống thành các phần riêng biệt:
    *   `apps/web`: Ứng dụng chính (Giao diện người dùng và API).
    *   `apps/unsubscriber`: Một dịch vụ riêng biệt dùng **Playwright** để tự động hóa việc hủy đăng ký (unsubscribe) trên web.
    *   `packages/`: Các thư viện dùng chung (CLI, logic tích hợp Resend, Loops, Tinybird).
*   **Kiến trúc AI-Centric:** Logic AI không chỉ là "chatbot" mà được nhúng sâu vào luồng xử lý dữ liệu:
    *   **Agentic Workflow:** Sử dụng AI để phân tích nội dung email, từ đó tự động chọn "Rule" (quy tắc) và "Action" (hành động) phù hợp.
    *   **MCP (Model Context Protocol):** Tích hợp các công cụ bên ngoài (HubSpot, Notion) để AI có thêm ngữ cảnh khi trả lời email.
*   **Kiến trúc hướng sự kiện (Event-driven):** Sử dụng **Google PubSub** và Webhooks để nhận thông báo email mới theo thời gian thực thay vì quét (polling) liên tục, giúp tiết kiệm tài nguyên.
*   **Security & Privacy:** Dữ liệu nhạy cảm được mã hóa (`EMAIL_ENCRYPT_SECRET`), hỗ trợ tự lưu trữ (Self-hosting) qua Docker để người dùng kiểm soát dữ liệu.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Prompt Engineering nâng cao:** Hệ thống sử dụng các kỹ thuật như *Few-shot prompting* và *Structured Output* (Zod schema) để đảm bảo AI trả về định dạng JSON chính xác cho các hành động (như tạo Rule, phân loại email).
*   **RAG (Retrieval-Augmented Generation) đơn giản:** Trích xuất kiến thức từ lịch sử email và kho tri thức (`Knowledge Base`) để AI soạn thảo phản hồi cá nhân hóa.
*   **Browser Automation:** Trong `apps/unsubscriber`, dự án dùng AI để "nhìn" vào trang web (DOM), xác định nút "Unsubscribe" và điều khiển Playwright thực hiện click tự động.
*   **SWR (Stale-While-Revalidate):** Kỹ thuật đồng bộ dữ liệu phía Client giúp giao diện phản hồi tức thì và hoạt động mượt mà ngay cả khi mạng chậm.
*   **Server Actions:** Thay vì dùng API route truyền thống cho mọi thứ, dự án sử dụng Next.js Server Actions cho các tác vụ thay đổi dữ liệu (mutations) để tăng tính bảo mật và giảm boilerplate code.

### 4. Tóm tắt luồng hoạt động (Operational Workflow)
1.  **Kết nối (Onboarding):** Người dùng đăng nhập qua Google/Microsoft OAuth. Hệ thống yêu cầu quyền truy cập (Scopes) để đọc/ghi email và lịch.
2.  **Đồng bộ & Theo dõi (Sync & Watch):** 
    *   Hệ thống thiết lập chế độ "Watch" qua PubSub/Webhooks. 
    *   Khi có email mới, Google/Microsoft gửi tín hiệu về API của Inbox Zero.
3.  **Xử lý AI (AI Processing):**
    *   **Phân loại:** Email được đưa qua AI để gắn nhãn (Newsletter, Marketing, Cold Email, v.v.).
    *   **Áp dụng quy tắc:** AI đối chiếu nội dung email với các "Cursor Rules" mà người dùng đã thiết lập.
    *   **Hành động:** Nếu khớp, AI thực hiện hành động tự động (Lưu trữ, Trả lời nháp, Forward, hoặc Chặn).
4.  **Tương tác người dùng:**
    *   Người dùng xem các bản thảo (Drafts) do AI viết sẵn.
    *   Sử dụng tính năng "Bulk Unsubscribe" để dọn dẹp hàng loạt các thư rác.
    *   Xem báo cáo Analytics về hiệu suất xử lý email.
5.  **Tự động hóa định kỳ (Cron Jobs):** Các tác vụ như làm mới quyền truy cập, gửi bản tin tóm tắt (Meeting Briefs) được thực hiện bởi một container `cron` riêng biệt.

### Kết luận
**Inbox Zero** là một dự án mã nguồn mở có kiến trúc rất bài bản, kết hợp giữa khả năng tự động hóa của AI và hiệu suất của Next.js. Đây không chỉ là một trình duyệt email mà là một **AI Agent** thực thụ giúp người dùng quản lý hòm thư bằng ngôn ngữ tự nhiên.