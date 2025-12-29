Dựa trên mã nguồn và cấu trúc thư mục của kho lưu trữ **Firecrawl**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này:

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

Firecrawl là một hệ thống **Polyglot** (đa ngôn ngữ) được thiết kế để tối ưu hóa hiệu suất cho từng tác vụ cụ thể:

*   **TypeScript (Node.js):** Đóng vai trò là "bộ não" điều phối (Orchestrator). Sử dụng **Express** để xây dựng API và **BullMQ/Redis** để quản lý hàng đợi công việc (Job Queue).
*   **Rust (Native Bridge):** Đây là điểm cực kỳ thú vị. Firecrawl sử dụng `napi-rs` để viết các module hiệu suất cao bằng Rust và gọi trực tiếp từ Node.js. Các tác vụ như:
    *   **Crawler logic:** Lọc link, xử lý `robots.txt`, sitemap.
    *   **HTML Transformation:** Dọn dẹp HTML, trích xuất metadata.
    *   **Document Conversion:** Chuyển đổi file Office (Docx, Xlsx, Odt, Rtf) sang HTML/Markdown.
*   **Go (Shared Library):** Sử dụng Go để xử lý chuyển đổi HTML sang Markdown (`go-html-to-md`) thông qua thư viện liên kết động (`.so`), giúp tận dụng tốc độ xử lý chuỗi cực nhanh của Go.
*   **Playwright:** Sử dụng để render các trang web nặng JavaScript (Single Page Applications - SPA) và vượt qua các cơ chế chống bot cơ bản.
*   **Cơ sở dữ liệu & Hạ tầng:**
    *   **Redis:** Lưu trữ hàng đợi, cache và giới hạn lưu lượng (Rate limiting).
    *   **PostgreSQL (với NUQ):** Theo dõi trạng thái Job, quản lý định danh và ghi log hệ thống.
    *   **RabbitMQ:** Xử lý các thông điệp nội bộ và xử lý lỗi (Poison pill handling).

---

### 2. Tư duy Kiến trúc (Architectural Design)

Firecrawl được xây dựng theo mô hình **Distributed Worker Architecture** (Kiến trúc Worker phân tán):

1.  **Monorepo Pattern:** Quản lý cả API, SDK (JS, Python, Rust) và các module native trong cùng một kho lưu trữ để dễ dàng đồng bộ hóa phiên bản.
2.  **Decoupled Scraper Engine:** Hệ thống tách biệt giữa logic cào dữ liệu và "Engine" thực thi. Tùy vào cấu hình, nó có thể chọn `fetch` (nhanh, nhẹ) hoặc `playwright` (nặng nhưng render được JS).
3.  **Hybrid Performance Strategy:** Thay vì viết 100% bằng Node.js (vốn yếu về tính toán nặng), các tác vụ CPU-bound (như phân tích cú pháp HTML, xử lý PDF/Office) được đẩy xuống tầng **Rust** và **Go**.
4.  **Job-based Processing:** Mọi yêu cầu Crawl/Batch đều là bất đồng bộ. API nhận yêu cầu -> Đẩy vào Redis -> Trả về `jobId` -> Worker xử lý -> Client polling (hỏi thăm) kết quả.
5.  **LLM-First Design:** Kiến trúc tập trung vào việc tạo ra dữ liệu "sạch" cho AI (Markdown, JSON Schema) thay vì chỉ lấy HTML thô.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **LLM Extraction (Beta):** Sử dụng các Provider như OpenAI, Anthropic, Gemini hoặc Ollama (tự chạy) để trích xuất dữ liệu có cấu trúc từ Markdown dựa trên Schema (Zod/Pydantic) hoặc Prompt tự do.
*   **Anti-Bot & Proxy:** Tích hợp cơ chế xoay vòng Proxy, giả lập Browser Headers và cơ chế "Stealth mode" thông qua Playwright để tránh bị chặn.
*   **Sitemap & Discovery:** Kỹ thuật duyệt đệ quy (Recursive crawling) kết hợp với phân tích Sitemap XML (xử lý bằng Rust) để tìm kiếm mọi ngóc ngách của website một cách nhanh nhất.
*   **Content Transformation:** Sử dụng các bộ lọc CSS selector để loại bỏ rác (ads, footer, nav) trước khi chuyển đổi sang Markdown, giúp giảm token khi đưa vào LLM.
*   **Action Orchestration:** Kỹ thuật cho phép người dùng định nghĩa các hành động như: `click`, `wait`, `scroll`, `input` trước khi thực hiện cào dữ liệu (tương tự như một automation bot).

---

### 4. Tóm tắt luồng hoạt động (Operation Flow)

Dưới đây là luồng đi của một yêu cầu từ lúc bắt đầu đến khi trả về kết quả:

1.  **Tiếp nhận (API Layer):**
    *   Người dùng gửi yêu cầu POST `/v2/scrape` hoặc `/v2/crawl` kèm theo URL và các tùy chọn (format, schema).
    *   Hệ thống kiểm tra xác thực (API Key) và giới hạn lưu lượng (Rate Limit).

2.  **Lập lịch (Queue Layer):**
    *   Yêu cầu được đóng gói thành một Job và đẩy vào **Redis (BullMQ)**.
    *   Nếu là `crawl`, hệ thống sẽ khởi tạo một tiến trình theo dõi tiến độ.

3.  **Thực thi (Worker Layer):**
    *   **Engine Selection:** Worker lấy Job ra và quyết định dùng Engine nào (Fetch/Playwright/Fire-engine).
    *   **Fetching:** Thực hiện tải nội dung trang web.
    *   **Processing (Rust/Go Layer):**
        *   Dọn dẹp HTML, chuyển đổi sang Markdown.
        *   Nếu có file (PDF/Docx), gọi module Rust để chuyển sang text.
    *   **AI Extraction (Optional):** Nếu người dùng yêu cầu trích xuất dữ liệu, Markdown sẽ được gửi đến LLM (OpenAI/Anthropic...) để lấy JSON.

4.  **Trả kết quả (Output Layer):**
    *   Kết quả được lưu vào Database/Cache.
    *   Nếu là Job Scrape đơn lẻ: Trả về JSON ngay lập tức.
    *   Nếu là Job Crawl: Người dùng gọi GET `/v2/crawl/{id}` để nhận danh sách kết quả sau khi Worker hoàn tất.

**Tóm lại:** Firecrawl không chỉ là một thư viện Scraper đơn thuần, mà là một **Data Pipeline** phức tạp kết hợp sức mạnh của Node.js (điều phối), Rust/Go (tốc độ xử lý nội dung) và LLM (hiểu nội dung).