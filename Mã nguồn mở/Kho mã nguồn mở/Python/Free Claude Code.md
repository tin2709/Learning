Dựa trên nội dung mã nguồn và tài liệu kiến trúc của dự án **Free Claude Code**, dưới đây là phân tích chi tiết về hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Dự án được xây dựng trên một nền tảng Python hiện đại, tập trung vào hiệu suất cao và tính bất đồng bộ:
*   **Ngôn ngữ:** Python 3.14 (sử dụng các tính năng mới nhất như `slots`, `StrEnum`).
*   **Framework Web:** **FastAPI** xử lý các yêu cầu API từ Claude Code CLI hoặc VSCode extension.
*   **Quản lý gói:** **Astral `uv`** - công cụ cực nhanh để quản lý dependency và môi trường ảo.
*   **Giao thức truyền tải:** **Server-Sent Events (SSE)** - cốt lõi của việc stream kết quả từ LLM về client.
*   **Xử lý bất đồng bộ:** Sử dụng triệt để `asyncio` cho cả mạng (httpx/aiohttp) và quản lý tiến trình (subprocess).
*   **Định danh & Kiểm tra kiểu:** Pydantic (V2) để validate dữ liệu và `ty` để check type chặt chẽ.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của dự án tuân theo mô hình **Transparent Proxy & Protocol Adapter**:

*   **Tính trung lập của Giao thức (Protocol Neutrality):** Toàn bộ logic nghiệp vụ bên trong sử dụng định dạng của Anthropic làm "ngôn ngữ chung". Mọi Provider (NVIDIA NIM, OpenAI, v.v.) đều phải có Adapter để dịch ngược về định dạng Anthropic.
*   **Phân lớp rõ rệt (Layered Architecture):**
    *   `api/`: Lớp giao tiếp, định tuyến và tối ưu hóa yêu cầu.
    *   `providers/`: Lớp chuyển đổi logic cho từng backend cụ thể.
    *   `core/anthropic/`: Chứa các "contract" (hợp đồng dữ liệu) và logic xử lý SSE dùng chung.
    *   `messaging/`: Tách biệt hoàn toàn logic của Bot (Discord/Telegram) khỏi logic của API.
*   **Nguyên tắc "Fast-path" (Tối ưu hóa phản hồi):** Hệ thống có khả năng nhận diện các yêu cầu "vô nghĩa" hoặc mang tính thủ tục (như kiểm tra quota, tạo tiêu đề) để trả lời ngay lập tức mà không tốn tài nguyên gọi API thật.
*   **Kiến trúc Tree-based Threading:** Trong phần messaging, các cuộc hội thoại được quản lý dưới dạng cây (tree) cho phép người dùng "fork" (nhánh) cuộc hội thoại bằng cách reply tin nhắn.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Heuristic Tool Parser:** Kỹ thuật xử lý các Model không hỗ trợ Tool-use chính thức bằng cách parse output dạng text và chuyển đổi chúng thành các cấu trúc `tool_use` của Anthropic.
*   **Thinking Block Conversion:** Tự động bắt các thẻ `<think>` hoặc trường `reasoning_content` từ DeepSeek/NIM để chuyển đổi thành block `thinking` chuẩn của Claude.
*   **Sliding Window Rate Limiting:** Kỹ thuật giới hạn tốc độ yêu cầu theo cửa sổ thời gian trượt (Strict Sliding Window) để đảm bảo không vi phạm chính sách của nhà cung cấp miễn phí (ví dụ: 40 req/min của NVIDIA NIM).
*   **Subprocess Management:** Quản lý vòng đời của tiến trình `claude` CLI thông qua một `ProcessRegistry`, đảm bảo dọn dẹp sạch sẽ (kill PID) khi server tắt để tránh tiến trình rác.
*   **Dependency Injection:** Sử dụng hệ thống DI của FastAPI để quản lý các Provider và Settings một cách linh hoạt.

### 4. Luồng hoạt động hệ thống (System Operation Flow)

1.  **Tiếp nhận (Ingest):** Claude Code gửi một POST request (thường là streaming) đến `/v1/messages`.
2.  **Xác thực & Định tuyến:** `require_api_key` kiểm tra token, sau đó `ModelRouter` quyết định Model này sẽ do Provider nào xử lý (ví dụ: yêu cầu Claude Sonnet được điều hướng sang DeepSeek Chat).
3.  **Tối ưu hóa (Optimization Check):** `try_optimizations` kiểm tra xem đây có phải là yêu cầu đơn giản không. Nếu đúng, trả về kết quả giả lập ngay.
4.  **Chuyển đổi yêu cầu (Request Translation):** 
    *   Nếu Provider dùng OpenAI format: `AnthropicToOpenAIConverter` sẽ dịch `messages` và `tools`.
    *   Nếu Provider dùng Native Anthropic: Dữ liệu được chuyển thẳng sau khi làm sạch.
5.  **Stream Processing:** 
    *   Proxy kết nối với Provider Upstream.
    *   Khi nhận được các chunk dữ liệu (OpenAI delta hoặc Anthropic event), Proxy sử dụng `SSEBuilder` để đóng gói lại đúng chuẩn Anthropic SSE.
    *   Các token "suy nghĩ" được xử lý riêng biệt để hiển thị đúng giao diện Claude.
6.  **Hoàn tất:** Trả về kết quả cuối cùng và đóng kết nối.

### Điểm đặc biệt:
Hệ thống này không chỉ là một proxy API đơn thuần mà còn là một **Agent Runner**. Thông qua Discord/Telegram, nó tự động khởi tạo các session CLI của Claude Code trong một không gian làm việc (`agent_workspace`), cho phép điều khiển lập trình từ xa thông qua giao diện chat mà vẫn giữ được toàn bộ sức mạnh của công cụ CLI gốc.