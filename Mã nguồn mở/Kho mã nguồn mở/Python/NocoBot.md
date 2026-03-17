Chào bạn, đây là bản phân tích chi tiết về hệ thống **NocoBot** dựa trên mã nguồn và cấu trúc thư mục bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technologies)

NocoBot là một hệ sinh thái hiện đại kết hợp giữa khả năng của LLM (Large Language Models) và giao thức MCP để tương tác với cơ sở dữ liệu.

*   **Ngôn ngữ lập trình:** Python 3.10+ (sử dụng `uv` làm trình quản lý gói để tối ưu tốc độ).
*   **Giao thức MCP (Model Context Protocol):** Sử dụng `FastMCP 3.0`. Đây là công nghệ then chốt giúp "đóng gói" các hàm của database thành các "công cụ" (tools) mà AI có thể hiểu và sử dụng được.
*   **Hệ thống Agent:** Sử dụng `LiteLLM` để kết nối với `OpenRouter` (Claude, GPT-4, ...), cho phép agent thực hiện suy luận đa bước (multi-step reasoning).
*   **Giao tiếp:** `python-telegram-bot` xử lý tương tác người dùng, tích hợp cơ chế async/await hoàn toàn.
*   **Hạ tầng:** Docker & Docker-compose, tối ưu hóa để triển khai trên các nền tảng PaaS như Dokploy.
*   **SDK Database:** Tự xây dựng bộ SDK Python cho NocoDB hỗ trợ Hybrid API (v2 cho Meta data và v3 cho Data operations).

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Hệ thống được thiết kế theo mô hình **Decoupled Agent Architecture** (Kiến trúc Agent tách rời):

*   **Phân tách Service (Monorepo):** Chia rõ ràng thành `nocodb` (SDK & MCP Server - đóng vai trò là "Cánh tay") và `nocobot` (Telegram UI & Agent - đóng vai trò là "Bộ não").
*   **Cơ chế Message Bus:** Sử dụng `MessageBus` với các hàng đợi `asyncio.Queue` để tách biệt luồng nhận tin (Inbound) và luồng gửi tin (Outbound). Điều này giúp Bot có thể xử lý các tác vụ AI tốn thời gian mà không làm treo giao diện Telegram.
*   **Kiến trúc Stateless MCP Server:** MCP Server có thể chạy độc lập, cho phép không chỉ NocoBot mà các ứng dụng khác (như Claude Desktop) cũng có thể kết nối và truy vấn dữ liệu.
*   **Security-First:** Tư duy "Default-deny" (từ chối mặc định). Chỉ những ID người dùng có trong danh sách trắng (allowlist) mới được phép tương tác.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Agent Loop (ReAct Pattern):** Agent không chỉ trả lời mà còn biết "suy nghĩ". Nếu người dùng hỏi một câu phức tạp, Agent sẽ tự động: Gọi tool liệt kê bảng -> Gọi tool xem schema -> Gọi tool truy vấn dữ liệu -> Tổng hợp kết quả.
*   **Two-Tier Truncation (Cắt tỉa dữ liệu 2 lớp):**
    *   *Lớp thực thi (Inference):* Cắt dữ liệu ở 4000 ký tự để không làm tràn ngữ cảnh LLM (Context window).
    *   *Lớp lưu trữ (History):* Cắt ở 500 ký tự để tiết kiệm bộ nhớ và token cho các lượt chat sau.
*   **Progressive Text Streaming:** Vì Telegram không hỗ trợ streaming nguyên bản như Web, Bot sử dụng kỹ thuật liên tục chỉnh sửa tin nhắn (`edit_message_text`) để tạo hiệu ứng văn bản hiện ra dần dần.
*   **Markdown Table Rendering:** Kỹ thuật chuyển đổi dữ liệu thô từ database thành bảng Markdown với định dạng Align (căn lề) chuẩn để hiển thị đẹp mắt trong thẻ `<pre>` của Telegram.
*   **Persistent MCP Sessions:** Duy trì kết nối MCP liên tục thay vì bắt tay (handshake) lại mỗi lần gọi tool, giúp giảm đáng kể độ trễ (latency).
*   **Rate Limiting (Token Bucket):** Sử dụng thuật toán thùng mã báo để giới hạn số lượng tin nhắn mỗi người dùng (mặc định 10 msg/60s), bảo vệ tài khoản LLM khỏi bị spam.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Tiếp nhận:** Người dùng gửi tin nhắn trên Telegram -> `TelegramChannel` kiểm tra quyền truy cập & Rate limit -> Đẩy tin nhắn vào `MessageBus` (Inbound).
2.  **Suy luận:** `AgentLoop` lấy tin nhắn từ Bus -> Kết hợp với lịch sử chat -> Gửi tới LLM qua `LiteLLM`.
3.  **Hành động (Tools):** LLM phân tích và yêu cầu gọi tool (ví dụ: `records_list`) -> Agent thông qua `MCPClient` gửi yêu cầu tới MCP Server.
4.  **Truy vấn:** MCP Server sử dụng SDK nội bộ gọi API thực tế của NocoDB -> Lấy dữ liệu về -> Trả ngược kết quả cho Agent.
5.  **Phản hồi:** Agent gửi kết quả tool trở lại LLM -> LLM viết câu trả lời bằng ngôn ngữ tự nhiên -> Đẩy vào `MessageBus` (Outbound).
6.  **Hiển thị:** `TelegramChannel` nhận tin nhắn Outbound -> Format Markdown/Table -> Stream kết quả cho người dùng qua Telegram.

**Điểm đặc biệt:** Toàn bộ quá trình Agent đang "suy nghĩ" hoặc "gọi tool", Bot sẽ gửi tín hiệu `typing` hoặc `uploading document` liên tục lên Telegram để người dùng không cảm thấy bị treo.