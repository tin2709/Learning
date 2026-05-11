Chào bạn, dựa trên mã nguồn và cấu trúc của dự án **Skyvern**, đây là một nền tảng **Browser-based Agentic Workflow** (Tự động hóa trình duyệt dựa trên tác tử AI) cực kỳ tiên tiến. Dự án này đại diện cho làn sóng "RPA 2.0" – nơi AI không chỉ thực hiện lệnh mà còn "nhìn" và "suy nghĩ" như con người.

Dưới đây là phân tích chi tiết:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Analysis)

Skyvern kết hợp sức mạnh của thị giác máy tính và mô hình ngôn ngữ lớn để giải quyết bài toán tự động hóa web mà không cần các selector (XPath/CSS) cứng nhắc:

*   **Computer Vision-driven Navigation:** Thay vì chỉ dựa vào mã nguồn HTML (thường bị thay đổi hoặc obfuscate), Skyvern chụp ảnh màn hình (screenshots) và sử dụng **Vision LLMs** (như GPT-4o, Claude 3.5 Sonnet) để xác định các thành phần tương tác.
*   **WebEye Engine:** Đây là "bộ não" thực thi lớp dưới, được xây dựng dựa trên **Playwright**. Nó bao gồm các module xử lý DOM phức tạp, giúp trích xuất các "semantic tree" (cây ngữ nghĩa) thay vì một rừng thẻ HTML thừa thãi.
*   **LiteLLM & Multi-provider Strategy:** Sử dụng **LiteLLM** làm lớp trừu tượng để gọi hơn 100 loại LLMs khác nhau (OpenAI, Anthropic, Gemini, Ollama), giúp hệ thống có tính linh hoạt cực cao và tránh bị khóa chặt vào một nhà cung cấp.
*   **Secure Credential Vault:** Tích hợp sâu với **Bitwarden** và **1Password** thông qua CLI Server để xử lý việc đăng nhập tự động mà không lưu mật khẩu trực tiếp trong script, đảm bảo tiêu chuẩn an ninh doanh nghiệp.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Skyvern được xây dựng theo mô hình **Agentic Orchestration**:

*   **Block-based Workflow Design:** Hệ thống chia nhỏ các hành động thành các "Block" (Navigation, Extraction, Validation, Loop, v.v.). Tư duy này cho phép người dùng xây dựng các luồng phức tạp như một biểu đồ DAG (Directed Acyclic Graph) thông qua UI.
*   **Stateful Browser Sessions:** Khác với các script automation thông thường sẽ đóng trình duyệt sau mỗi task, Skyvern hỗ trợ **Persistent Browser Sessions**. Trạng thái trình duyệt (Cookies, LocalStorage) được duy trì qua nhiều bước chạy của Workflow, cho phép xử lý các phiên làm việc kéo dài nhiều ngày.
*   **Hybrid Storage:** Kiến trúc sử dụng **PostgreSQL** (thông qua SQLAlchemy/Alembic) để lưu trữ cấu trúc workflow và lịch sử chạy, trong khi các **Artifacts** (video quay màn hình, file tải về, log chi tiết) được quản lý riêng biệt để tối ưu hiệu năng.
*   **Planner-Executor Pattern:** Hệ thống hoạt động theo vòng lặp: **Observe** (Nhìn màn hình) -> **Plan** (Lập kế hoạch hành động tiếp theo) -> **Execute** (Thực thi qua Playwright) -> **Validate** (Kiểm tra kết quả).

---

### 3. Kỹ thuật Lập trình Đặc sắc (Distinctive Programming Techniques)

*   **State Machine cho Browser Recording:** Trong module `services/browser_recording`, Skyvern sử dụng các máy trạng thái (State Machines) để ghi lại hành động của người dùng (click, hover, input) và chuyển đổi chúng thành các Block logic trong Workflow một cách mượt mà.
*   **Semantic Locators:** Thay vì `page.click("#id-123")`, Skyvern cho phép code kiểu `page.click(prompt="nút đăng ký màu xanh")`. Kỹ thuật này sử dụng LLM để mapping từ ngôn ngữ tự nhiên sang tọa độ hoặc phần tử DOM chính xác tại thời điểm thực thi.
*   **Jinja2 Dynamic Prompting:** Dự án sử dụng **Jinja2 templates** cho các prompt gửi tới LLM. Điều này cho phép nhúng các biến môi trường, dữ liệu trích xuất từ bước trước vào prompt một cách linh hoạt, tạo ra khả năng "suy luận theo ngữ cảnh".
*   **Soft Delete & Resource Cleanup:** Việc triển khai `_soft_delete.py` và các `cleanup_service` cho thấy sự đầu tư vào quản lý tài nguyên, đặc biệt quan trọng khi chạy hàng trăm container trình duyệt cùng lúc.

---

### 4. Luồng Hoạt động Hệ thống (System Operation Flow)

Luồng hoạt động của một **Workflow Run** diễn ra như sau:

1.  **Trigger & Context Injection:** Người dùng kích hoạt workflow (qua API/UI/Schedules). Các tham số (parameters) và thông tin đăng nhập từ Vault được nạp vào context.
2.  **Browser Orchestration:** `BrowserManager` khởi tạo một instance (headless hoặc headful). Nếu có cấu hình Proxy (Residential/Mobile), nó sẽ được áp dụng ngay tại bước này.
3.  **The Agent Loop:**
    *   **Scraping:** `webeye` quét trang web, chụp ảnh và nén dữ liệu DOM thành dạng rút gọn (Compressed DOM).
    *   **Reasoning:** Dữ liệu này được gửi tới LLM cùng với mục tiêu của Block hiện tại. LLM trả về hành động JSON (ví dụ: `{ "action": "click", "element": "search_button" }`).
    *   **Acting:** Playwright thực thi hành động. Nếu gặp CAPTCHA, hệ thống sẽ gọi các solver tích hợp hoặc yêu cầu Human-in-the-loop (nếu cần).
4.  **Data Exfiltration & Artifact Bundling:** Kết quả (dữ liệu JSON trích xuất được hoặc file) được kiểm tra (Validation). Toàn bộ quá trình được quay video và lưu lại thành Artifact.
5.  **Status Sync:** Trạng thái của từng bước được đồng bộ thời gian thực về Frontend qua **Event Source Stream (SSE)** để người dùng theo dõi màn hình livestream.

### Tổng kết
Skyvern không chỉ là một công cụ automation, mà là một **Operating System cho AI Agents** để tương tác với thế giới web. Nó giải quyết triệt để vấn đề "brittle automation" (tự động hóa dễ gãy) bằng cách thay thế các selector cứng bằng khả năng hiểu giao diện (UI Understanding) của mô hình thị giác. Đây là kiến trúc lý tưởng cho các bài toán RPA quy mô lớn và phức tạp.