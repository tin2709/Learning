Dưới đây là phân tích chi tiết về dự án **Fli (Flight Search MCP Server and Library)** dựa trên mã nguồn và tài liệu bạn cung cấp. Dự án này là một ví dụ điển hình về kỹ thuật **Reverse Engineering API** hiện đại và cách xây dựng công cụ cho hệ sinh thái AI (MCP).

---

### 1. Công nghệ cốt lõi (Technology Core)

Dự án tận dụng các thư viện Python hiện đại nhất để giải quyết các bài toán phức tạp:

*   **curl_cffi (Trọng tâm):** Đây là "vũ khí" chính. Thay vì dùng `requests` hay `httpx` thông thường, `curl_cffi` cho phép giả lập **TLS Fingerprint** của các trình duyệt như Chrome. Điều này cực kỳ quan trọng khi giao tiếp với API nội bộ của Google để tránh bị chặn bởi các hệ thống chống bot (Anti-bot).
*   **FastMCP:** Sử dụng framework để triển khai **Model Context Protocol**. Đây là tiêu chuẩn mới giúp các mô hình ngôn ngữ lớn (LLM) như Claude có thể sử dụng các công cụ ngoại vi một cách an toàn và chuẩn hóa.
*   **Pydantic (v2):** Toàn bộ cấu trúc dữ liệu, từ filter tìm kiếm đến kết quả trả về, đều được định nghĩa qua Pydantic. Điều này đảm bảo dữ liệu luôn đúng kiểu (type-safe) và tự động validate lỗi ngay từ đầu vào.
*   **Typer & Rich:** Xây dựng CLI mạnh mẽ. `Typer` xử lý logic lệnh, còn `Rich` xử lý hiển thị bảng biểu, màu sắc và biểu đồ (plotext) ngay trên terminal.
*   **Tenacity & Ratelimit:** Đảm bảo hệ thống hoạt động ổn định thông qua cơ chế tự động thử lại (retry) với độ trễ tăng dần (exponential backoff) và giới hạn tần suất gọi API (10 req/s) để tránh bị khóa IP.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Fli được thiết kế theo hướng **Modular (Mô-đun hóa)** và **Contract-driven (Dựa trên hợp đồng dữ liệu)**:

*   **Separation of Concerns (Phân tách trách nhiệm):**
    *   `fli/models/`: Định nghĩa "hình dạng" dữ liệu (Data Schema).
    *   `fli/core/`: Chứa các bộ logic dùng chung (Parsers, Builders). Đây là cầu nối giữa Interface (CLI/MCP) và Engine.
    *   `fli/search/`: Engine thực thi việc gọi API và xử lý phản hồi.
    *   `fli/cli/` & `fli/mcp/`: Các giao diện tương tác với người dùng/AI.
*   **Shared Core Strategy:** Cả CLI và MCP Server đều dùng chung các hàm logic trong `fli/core/`. Điều này đảm bảo dù bạn tìm chuyến bay bằng dòng lệnh hay nhờ Claude tìm, kết quả và cách xử lý tham số là hoàn toàn giống nhau.
*   **Data-Driven Enums:** Thay vì fix cứng mã sân bay/hãng hàng không, dự án có script (`generate_enums.py`) để tạo tự động các Class Enum từ file CSV. Cách tiếp cận này giúp code sạch hơn, hỗ trợ auto-complete tốt và dễ cập nhật dữ liệu hàng tuần.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **API Reverse Engineering (Kỹ thuật then chốt):** Dự án không cào dữ liệu (scraping) từ HTML. Thay vào đó, nó giải mã cấu trúc JSON lồng nhau cực kỳ phức tạp của Google Flights (thường là các mảng lồng nhau kiểu Protobuf-over-JSON).
    *   Xem file `fli/models/google_flights/flights.py`: Hàm `format()` chuyển đổi các Object Python thành các mảng số và chuỗi theo đúng thứ tự mà API nội bộ của Google yêu cầu.
*   **Impersonation Logic:** Trong `fli/search/client.py`, client được cấu hình để "đóng vai" một trình duyệt thực thụ. Điều này bao gồm việc quản lý Header, Cookie và đặc biệt là cách bắt tay TLS.
*   **Robust Validation (Validation đa tầng):** Sử dụng `@field_validator` và `@model_validator` để kiểm tra logic nghiệp vụ phức tạp. Ví dụ: Ngày về phải sau ngày đi, sân bay đi phải khác sân bay đến, giới hạn số lượng hành khách...
*   **Singleton Client:** Sử dụng một thực thể Client duy nhất (`get_client`) để quản lý Session, giúp tiết kiệm tài nguyên và duy trì trạng thái kết nối hiệu quả.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một yêu cầu tìm kiếm diễn ra như sau:

1.  **Input (Đầu vào):** Người dùng nhập lệnh CLI `fli flights JFK LHR 2026-03-15` hoặc LLM gọi tool `search_flights`.
2.  **Parsing (Phân tích):** `fli/core/parsers.py` sẽ chuyển đổi các chuỗi thô (JFK, LHR) thành các Enum (`Airport.JFK`, `Airport.LHR`).
3.  **Model Construction:** Dữ liệu được nạp vào `FlightSearchFilters` (Pydantic). Tại đây, mọi ràng buộc dữ liệu được kiểm tra.
4.  **Encoding:** Hàm `encode()` trong model thực hiện việc "đóng gói" dữ liệu thành một chuỗi URL-encoded JSON phức tạp (đây là bước "ma thuật" đã được reverse engineer).
5.  **Request:** `SearchFlights` gửi yêu cầu POST thông qua `curl_cffi` Client đến endpoint của Google.
6.  **Decoding:** Phản hồi từ Google (thường có tiền tố chống hijack JSON như `)]}'`) được cắt bỏ và phân tích cú pháp.
7.  **Mapping:** Dữ liệu mảng thô được ánh xạ ngược lại vào model `FlightResult` và `FlightLeg`.
8.  **Output:** Kết quả được trả về dưới dạng JSON cho AI hoặc hiển thị bảng biểu đẹp mắt cho người dùng CLI.

---

### Tổng kết
Fli không chỉ là một thư viện tìm chuyến bay; nó là một bài học về **Python Engineering**. Dự án cho thấy cách xử lý các API "khó nhằn" bằng cách giả lập trình duyệt ở mức thấp, đồng thời trình bày dữ liệu thông qua các giao thức hiện đại nhất hiện nay (MCP).

**Điểm mạnh nhất:** Sự ổn định nhờ không dùng Web Scraping (HTML) và khả năng tích hợp cực nhanh vào các AI Agent.