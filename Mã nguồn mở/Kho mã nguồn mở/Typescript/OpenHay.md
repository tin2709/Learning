Chào bạn, đây là bản phân tích chi tiết về kiến trúc và kỹ thuật của dự án **OpenHay** – một nền tảng tìm kiếm và nghiên cứu bằng AI (tương tự Perplexity/AIHay) được tối ưu cho tiếng Việt.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một "stack" công nghệ rất hiện đại, tập trung vào hiệu suất xử lý ngôn ngữ và khả năng mở rộng của Agent:

*   **Backend (Python 3.13):**
    *   **FastAPI:** Framework web hiệu năng cao, xử lý bất đồng bộ (async).
    *   **PydanticAI:** Thư viện then chốt để xây dựng và điều phối các AI Agent (Agentic Orchestration).
    *   **PostgreSQL + SQLModel/SQLAlchemy:** Quản lý dữ liệu quan hệ và mô hình hóa dữ liệu (ORM).
    *   **Crawl4AI:** Công cụ trích xuất nội dung web mạnh mẽ, hỗ trợ crawl sâu và chuyển đổi sang Markdown.
    *   **LLM Providers:** Chủ yếu dùng Google Gemini (model `gemini-3-flash-preview` được ưu tiên vì tốc độ và chi phí).
*   **Frontend (TypeScript):**
    *   **React + Vite:** Xây dựng giao diện SPA nhanh, mượt.
    *   **Tailwind CSS + shadcn/ui:** Hệ thống giao diện đồng nhất, chuyên nghiệp.
*   **Dịch vụ bên thứ ba:**
    *   **Brave Search API:** Công cụ tìm kiếm web cung cấp dữ liệu đầu vào cho AI.
    *   **Logfire:** Quan sát (Observability) và truy vết (tracing) hoạt động của Agent.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của OpenHay được thiết kế theo mô hình **Layered Architecture (Kiến trúc phân lớp)** và **Agent-Centric (Lấy Agent làm trung tâm)**:

*   **Phân lớp chức năng:**
    *   *Routers:* Tiếp nhận yêu cầu HTTP/SSE.
    *   *Services:* Xử lý logic nghiệp vụ (quản lý hội thoại, giải mã media).
    *   *Agents:* "Bộ não" thực hiện các tác vụ suy luận và sử dụng công cụ (tools).
    *   *Repositories:* Tương tác với cơ sở dữ liệu.
*   **Hệ thống Multi-Agent phân cấp (Hierarchical):** Đặc biệt trong tính năng "Deep Research", kiến trúc chia làm:
    *   **Lead Agent:** Lập kế hoạch, chia nhỏ nhiệm vụ và tổng hợp báo cáo.
    *   **Sub-agents:** Chạy song song để tìm kiếm và trích xuất dữ liệu chi tiết.
    *   **Citation Agent:** Chuyên biệt hóa nhiệm vụ trích dẫn nguồn để đảm bảo tính xác thực.
*   **Tư duy Stateless & Streaming:** Hệ thống ưu tiên phản hồi thời gian thực qua Server-Sent Events (SSE), cho phép người dùng thấy AI "suy nghĩ" và "viết" từng chữ thay vì đợi phản hồi một lần.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý luồng SSE (Server-Sent Events):** Kỹ thuật định dạng dữ liệu theo chuẩn `event: type \n data: payload \n\n` để truyền tải đồng thời cả nội dung chat, suy nghĩ của Agent (thinking), và kết quả tìm kiếm web.
*   **Kỹ thuật Trích dẫn (Citation Integration):** Sử dụng một Agent riêng để đọc báo cáo và danh sách URL đã fetch, sau đó chèn các thẻ trích dẫn `[n]` vào văn bản một cách logic mà không làm biến đổi nội dung gốc.
*   **Crawl sâu và Pruning Content:** Sử dụng chiến lược BFS (Breadth-First Search) để crawl nhiều trang con của một trang tin tức, kết hợp với `PruningContentFilter` để loại bỏ rác (newsletter, footer, ads), chỉ giữ lại nội dung hữu ích cho AI.
*   **Quản lý Quota & Rate Limiting:** Tích hợp middleware kiểm soát số lượng yêu cầu theo IP và giới hạn RPM (Requests Per Minute) cho từng model AI (Gemini Flash vs Pro) để tránh bị khóa API.
*   **RAG (Retrieval-Augmented Generation) liên tục:** Agent không chỉ dùng kiến thức tĩnh mà liên tục gọi công cụ `search_web` và `fetch_url_content` để cập nhật dữ liệu mới nhất từ Internet trước khi trả lời.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Một yêu cầu nghiên cứu (Deep Research) điển hình sẽ diễn ra như sau:

1.  **Nhận yêu cầu:** Người dùng gửi query qua `/api/research`.
2.  **Khởi tạo hội thoại:** Backend kiểm tra JWT, tạo/truy xuất Conversation trong DB và mở kết nối SSE.
3.  **Lập kế hoạch (Lead Agent):** Lead Agent nhận query, phân tích xem đây là dạng tìm kiếm rộng hay sâu, sau đó tạo ra một danh sách các "Prompts" cho các Sub-agents.
4.  **Thực thi song song (Sub-agents):** Các Sub-agents đồng loạt gọi Brave Search, sau đó dùng Crawl4AI để đọc nội dung chi tiết của các trang web phù hợp nhất.
5.  **Trích dẫn & Tổng hợp:**
    *   Các Sub-agents trả về các mẩu báo cáo nhỏ.
    *   **Citation Agent** kiểm tra chéo nguồn và chèn số hiệu trích dẫn.
    *   Lead Agent nhận dữ liệu đã có trích dẫn để viết thành một báo cáo hoàn chỉnh.
6.  **Phản hồi & Lưu trữ:** Nội dung được stream về client qua SSE. Toàn bộ lịch sử (bao gồm cả các bản tin JSON của PydanticAI) được lưu vào bảng `conversation_message_run` để phục vụ các lần hỏi tiếp theo (Context persistence).

**Đánh giá:** Dự án này là một tài liệu tham khảo tuyệt vời cho việc xây dựng ứng dụng AI thế hệ mới (Agentic Apps) thay vì chỉ là các ứng dụng RAG đơn giản. Việc sử dụng PydanticAI giúp code backend cực kỳ sạch sẽ và dễ bảo trì.