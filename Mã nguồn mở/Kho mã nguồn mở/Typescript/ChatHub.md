Dựa trên mã nguồn của dự án **ChatHub**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật mà dự án này sử dụng:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Dự án được xây dựng trên một ngăn xếp công nghệ hiện đại dành cho Browser Extension:

*   **Framework & Ngôn ngữ:** React 18, TypeScript (chiếm 97.8%).
*   **Build Tool:** Vite kết hợp với `@crxjs/vite-plugin` (giúp việc phát triển extension nhanh hơn với Hot Module Replacement).
*   **Quản lý trạng thái (State Management):** 
    *   **Jotai:** Sử dụng kiến trúc "Atomic State" để quản lý trạng thái cực kỳ chi tiết (ví dụ: mỗi khung chat là một atom riêng biệt).
    *   **Immer:** Kết hợp với Jotai để xử lý các state phức tạp (như mảng tin nhắn) một cách bất biến (immutable).
*   **Định tuyến (Routing):** **TanStack Router** – một thư viện routing mạnh mẽ, hỗ trợ type-safe tuyệt đối cho các URL.
*   **Giao diện (UI/Styling):** Tailwind CSS, Framer Motion (cho animation), Headless UI (cho các component như Dialog, Tabs).
*   **Xử lý Dữ liệu/Network:** 
    *   **SWR:** Dùng để fetch dữ liệu từ server-api và cache kết quả.
    *   **Ofetch:** Một thư viện fetch HTTP cải tiến, hỗ trợ tốt cho môi trường trình duyệt.
    *   **WebExtension Polyfill:** Đảm bảo extension chạy tốt trên cả Chrome, Edge và Firefox.

### 2. Tư duy kiến trúc (Architectural Thinking)

Dự án áp dụng tư duy **"Abstraction Layer"** (Lớp trừu tượng) rất rõ rệt:

*   **Bot Abstraction (src/app/bots/):**
    *   Tất cả các chatbot (ChatGPT, Claude, Bing, v.v.) đều kế thừa từ một lớp trừu tượng `AbstractBot`. 
    *   Điều này cho phép giao diện người dùng tương tác với mọi bot thông qua một giao thức chung (`sendMessage`, `resetConversation`), bất kể bot đó sử dụng API chính thức, WebApp Scraping hay GraphQL.
*   **Context Isolation:** Sử dụng Background Scripts để quản lý các tác vụ chạy ngầm (như xử lý cookie Twitter/Grok) và Content Scripts để "lách" qua các rào cản bảo mật của các trang web chatbot gốc (như OpenAI).
*   **Multi-layout System:** Kiến trúc hỗ trợ nhiều bố cục (All-in-one 2x2, 3x3, hoặc đơn lẻ) bằng cách sử dụng `atomFamily` trong Jotai để tạo ra các thực thể bot độc lập chạy song song mà không bị xung đột dữ liệu.

### 3. Các kỹ thuật chính (Key Techniques)

Đây là phần thú vị nhất về mặt kỹ thuật của ChatHub:

*   **Proxy Fetching (src/services/proxy-fetch.ts):** Extension tạo ra một tab ẩn (pinned tab) của ChatGPT, sau đó dùng `Browser.runtime.sendMessage` để yêu cầu tab đó thực hiện các fetch thay mặt extension. Kỹ thuật này giúp vượt qua lỗi CORS và tận dụng phiên đăng nhập (session) của người dùng.
*   **Streaming & SSE (src/utils/sse.ts):** Sử dụng `eventsource-parser` và `ReadableStream` để xử lý phản hồi dạng stream từ AI. Điều này tạo ra hiệu ứng chữ hiện ra dần dần (typing effect) mượt mà.
*   **Header Manipulation (src/rules/):** Sử dụng **Declarative Net Request (DNR)** của Manifest V3 để sửa đổi Header (Origin, Referer, User-Agent) của các yêu cầu mạng. Đây là cách ChatHub "giả danh" là trình duyệt chính chủ để gọi API của Bing hoặc Baichuan.
*   **Web Access Agent (src/services/agent/):** Một hệ thống "tác nhân" mini. Trước khi gửi câu hỏi cho AI, nó sẽ thực hiện tìm kiếm trên DuckDuckGo/Bing News, trích xuất nội dung trang web, và nhúng vào prompt (RAG đơn giản) để AI có thông tin cập nhật nhất.
*   **Bảo mật & Cookies (src/background/twitter-cookie.ts):** Tự động mở tab ẩn danh để trích xuất `csrf-token` từ cookies của các dịch vụ như Twitter để hỗ trợ bot Grok.

### 4. Tóm tắt luồng hoạt động (Workflow)

1.  **Khởi tạo:** Người dùng mở extension -> `main.tsx` chạy -> TanStack Router xác định trang (All-in-one hoặc Single bot).
2.  **Quản lý Bot:** `useChat` hook khởi tạo bot tương ứng thông qua `createBotInstance`.
3.  **Gửi tin nhắn:**
    *   User nhập liệu -> `ChatMessageInput`.
    *   Nếu bật **Web Access**, `Agent` sẽ chạy: Search Web -> Lấy ngữ cảnh -> Tạo Prompt mới.
    *   Bot cụ thể (ví dụ: `ChatGPTWebBot`) nhận Prompt.
4.  **Xử lý Network:** 
    *   Bot gọi `proxyFetch` (nếu là web mode) hoặc `fetch` (nếu là API mode). 
    *   DNR Rules tự động gắn thêm các header cần thiết để lách kiểm tra của server.
5.  **Phản hồi:** Server trả về dòng dữ liệu (Stream) -> `parseSSEResponse` xử lý -> Cập nhật vào Jotai Atom -> React Re-render tin nhắn trên màn hình.
6.  **Lưu trữ:** Tin nhắn được tự động đồng bộ vào `chrome.storage.local` thông qua các service trong `src/services/chat-history.ts`.

**Đánh giá:** Đây là một dự án có kỹ thuật cao về **Browser Extension**, đặc biệt là trong việc xử lý giao tiếp giữa các context và "vượt rào" các hạn chế của các nền tảng AI lớn.