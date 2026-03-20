Dựa trên nội dung mã nguồn và tài liệu bạn cung cấp về dự án **Daily Stock Analysis (DSA)**, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, ưu tiên khả năng mở rộng và hiệu suất xử lý dữ liệu:

*   **Ngôn ngữ & Framework:**
    *   **Python 3.10+:** Tận dụng Type Hinting mạnh mẽ và xử lý bất đồng bộ (`asyncio`).
    *   **FastAPI:** Framework web hiện đại, hiệu suất cao, tự động hóa tài liệu API (Swagger).
    *   **React + TypeScript + Vite:** Frontend nhanh, kiểu dữ liệu an toàn, trải nghiệm người dùng mượt mà.
    *   **Electron:** Đóng gói ứng dụng thành phần mềm Desktop chạy đa nền tảng.
*   **Trí tuệ nhân tạo (AI/LLM):**
    *   **LiteLLM:** Một lớp trừu tượng (abstraction layer) cực kỳ quan trọng, cho phép hệ thống gọi hàng chục loại LLM khác nhau (OpenAI, Gemini, Claude, DeepSeek, Ollama) thông qua một định dạng chung.
    *   **Multi-Agent Orchestration:** Sử dụng mô hình nhiều Agent chuyên biệt (Technical, Intel, Risk, Decision) để thực hiện các nhiệm vụ phức tạp thay vì một lời nhắc (prompt) duy nhất.
*   **Dữ liệu Tài chính:**
    *   **Đa nguồn (Multi-source):** Kết hợp AkShare, Tushare, YFinance, Baostock và TickFlow để đảm bảo dữ liệu luôn sẵn sàng ngay cả khi một nguồn bị lỗi.
    *   **Search Engine API:** Sử dụng Tavily, Bocha, Brave Search để tìm kiếm tin tức thị trường thời gian thực (RAG - Retrieval-Augmented Generation).
*   **Lưu trữ & Hạ tầng:**
    *   **SQLite:** Cơ sở dữ liệu nhẹ, không cần cấu hình phức tạp nhưng đủ mạnh cho nhu cầu cá nhân.
    *   **SQLAlchemy:** ORM giúp tương tác với DB một cách trừu tượng.
    *   **Docker:** Container hóa toàn bộ ứng dụng để triển khai dễ dàng trên mọi môi trường.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của DSA đi theo hướng **Modular & Pipeline-based**, tập trung vào tính linh hoạt và độ tin cậy:

*   **Kiến trúc Phân lớp (Layered Architecture):**
    *   *API Layer:* Giao diện RESTful cho Web/App.
    *   *Service Layer:* Chứa logic nghiệp vụ (phân tích, quản lý danh mục, xử lý Agent).
    *   *Repository Layer:* Trừu tượng hóa việc truy cập dữ liệu (DB).
    *   *Data Provider Layer:* Các Adapter chuyển đổi dữ liệu từ các nguồn tài chính khác nhau về định dạng chuẩn của hệ thống.
*   **Cấu hình hướng đối tượng (Object-Oriented Configuration):** Hệ thống quản lý hàng trăm biến môi trường thông qua một `ConfigManager` trung tâm, hỗ trợ nạp lại cấu hình (reload) ngay khi đang chạy mà không cần khởi động lại.
*   **Chiến lược Fallback (Dự phòng):** Đây là tư duy cốt lõi trong tài chính. Nếu nguồn dữ liệu ưu tiên 1 lỗi, hệ thống tự động chuyển sang nguồn 2. Tương tự với AI, nếu DeepSeek quá tải, hệ thống có thể chuyển sang Gemini hoặc Claude.
*   **Fail-open Design:** Đối với dữ liệu cơ bản (fundamentals), nếu việc thu thập thất bại, hệ thống vẫn tiếp tục quy trình phân tích kỹ thuật thay vì dừng toàn bộ (hard fail), giúp tối ưu tỷ lệ hoàn thành tác vụ.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

Mã nguồn thể hiện trình độ kỹ thuật cao thông qua các kỹ thuật:

*   **Asynchronous & Threading:** Sử dụng `async/await` cho các tác vụ I/O (gọi API, fetch dữ liệu) và `run_in_executor` để chạy các tác vụ tính toán nặng hoặc các thư viện đồng bộ trong môi trường bất đồng bộ mà không làm nghẽn Event Loop.
*   **Dependency Injection (DI):** Trong FastAPI, các thành phần như Database Session hoặc Configuration được tiêm (inject) vào các endpoint thông qua `Depends`, giúp mã nguồn dễ kiểm thử (testable).
*   **Schema Validation:** Sử dụng Pydantic để định nghĩa chặt chẽ cấu trúc dữ liệu đầu vào/đầu ra cho API, đảm bảo tính toàn vẹn dữ liệu từ Frontend đến Backend.
*   **Task Queue & SSE:** Hệ thống triển khai một hàng đợi tác vụ (`TaskQueue`) nội bộ. Khi người dùng yêu cầu phân tích, một tác vụ được đưa vào hàng đợi và trạng thái được đẩy về Frontend thông qua **Server-Sent Events (SSE)** để cập nhật tiến độ thời gian thực (ví dụ: "Đang lấy dữ liệu...", "AI đang suy nghĩ...").
*   **Template Engine (Jinja2):** Sử dụng Jinja2 để render các báo cáo Markdown phức tạp, cho phép tùy biến ngôn ngữ (Đa ngôn ngữ: Trung, Anh, Việt) và định dạng thông báo cho từng kênh (Telegram, Email, WeChat).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình phân tích một cổ phiếu diễn ra theo các bước chặt chẽ:

1.  **Trình kích hoạt (Trigger):** Người dùng nhập mã cổ phiếu (hoặc hệ thống chạy định kỳ qua GitHub Actions/Cron).
2.  **Thu thập dữ liệu (Data Ingestion):**
    *   Lấy giá lịch sử & thời gian thực (K-line).
    *   Thu thập dữ liệu cơ bản (P/E, ROE, cổ tức).
    *   Tìm kiếm tin tức và tâm lý mạng xã hội (Social Sentiment).
3.  **Xử lý qua Agent (Agentic Chain):**
    *   **Technical Agent:** Tính toán chỉ số kỹ thuật (MA, RSI, MACD), nhận diện mô hình nến.
    *   **Intel Agent:** Tổng hợp tin tức, lọc các sự kiện trọng yếu (Catalysts).
    *   **Risk Agent:** Kiểm tra các quy tắc kỷ luật giao dịch (Ví dụ: Không mua khi giá quá xa đường MA20 - BIAS quá cao).
    *   **Decision Agent:** Tổng hợp thông tin từ 3 Agent trên để đưa ra điểm mua/bán và điểm dừng lỗ.
4.  **Tạo báo cáo (Report Generation):** Dữ liệu được đưa vào Template Jinja2 để tạo ra báo cáo Markdown. Nếu kênh nhận không hỗ trợ Markdown, hệ thống sử dụng `imgkit` để chuyển báo cáo thành hình ảnh.
5.  **Phân phối (Distribution):** Gửi thông báo qua các kênh đã cấu hình (Webhook) và lưu trữ kết quả vào Database để người dùng xem lại trên Web UI.
6.  **Hồi quy (Backtest - Tùy chọn):** AI tự động đánh giá độ chính xác của các dự đoán trong quá khứ dựa trên biến động giá thực tế sau đó.

### Kết luận
Dự án này là một ví dụ điển hình về việc kết hợp giữa **Kỹ thuật phần mềm truyền thống** (hệ thống xử lý dữ liệu tin cậy) và **Kỹ thuật AI hiện đại** (Agentic Workflow). Điểm mạnh nhất nằm ở khả năng **trừu tượng hóa dữ liệu** và **quản lý cấu hình linh hoạt**, cho phép người dùng phổ thông cũng có thể vận hành một hệ thống phân tích định lượng phức tạp.