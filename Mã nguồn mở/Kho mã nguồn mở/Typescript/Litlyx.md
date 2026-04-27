Dựa trên mã nguồn của dự án **Litlyx**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật lập trình và luồng hoạt động của hệ thống.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Hệ thống được xây dựng trên một Stack hiện đại, tối ưu cho việc xử lý dữ liệu thời gian thực:

*   **Ngôn ngữ chính:** **TypeScript** (chiếm >70%), đảm bảo tính chặt chẽ và an toàn về kiểu dữ liệu cho toàn bộ hệ thống.
*   **Frontend (Dashboard):** **Nuxt 3 (Vue 3)**, sử dụng **Tailwind CSS** và **Shadcn UI** để xây dựng giao diện. Quản lý trạng thái bằng **Pinia**.
*   **Backend & Microservices:**
    *   **Producer/Consumer:** Node.js thuần (Express) để tối ưu hiệu suất thu thập dữ liệu.
    *   **tRPC:** Sử dụng để giao tiếp nội bộ giữa các dịch vụ (Email, Payments) với Dashboard một cách type-safe.
*   **Lưu trữ & Truy vấn:**
    *   **MongoDB (Primary DB):** Lưu trữ dữ liệu thô (Visits, Events) và dữ liệu người dùng.
    *   **Redis (Stream & Cache):** Đóng vai trò làm hàng đợi (Message Queue) thông qua **Redis Streams** và làm bộ nhớ đệm (Caching) cho các truy vấn phân tích nặng.
*   **AI Integration:** **OpenAI API** (GPT-4o-mini, GPT-5-nano) được tích hợp sâu để phân tích dữ liệu và cung cấp "Insight" cho người dùng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án Litlyx áp dụng mô hình **Event-Driven Microservices** kết hợp với **Monorepo**:

*   **Tách biệt Producer và Consumer:**
    *   **Producer:** Chỉ làm nhiệm vụ tiếp nhận HTTP request từ script tracking, kiểm tra whitelist domain/IP và đẩy dữ liệu vào Redis Stream. Điều này giúp Producer có khả năng chịu tải cực cao (High Throughput).
    *   **Consumer:** Đọc dữ liệu từ Redis Stream theo lô (batch) và ghi vào MongoDB. Việc tách biệt này giúp bảo vệ cơ sở dữ liệu khỏi bị treo khi traffic tăng đột biến.
*   **Kiến trúc Shared-Global:** Thư mục `shared_global` chứa toàn bộ Schema Mongoose, Utility và Logic dùng chung. Các service con (dashboard, producer, consumer) chỉ cần "copy" hoặc link tới đây để đảm bảo tính đồng nhất (Single Source of Truth).
*   **Thiết kế hướng Cung cấp Dịch vụ (Service-Oriented):** Các module như `payments` và `emails` hoạt động như các service độc lập, cung cấp API qua tRPC.
*   **Privacy-First (Không Cookie):** Hệ thống tạo `sessionHash` dựa trên việc băm (hash) địa chỉ IP + User Agent + Salt hàng ngày, cho phép theo dõi người dùng duy nhất mà không cần lưu cookie, tuân thủ GDPR.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Xử lý Aggregation Pipeline phức tạp:** Mã nguồn chứa rất nhiều truy vấn MongoDB Aggregation cực kỳ tinh xảo để tính toán: Bouncing rate, Session duration, Timeframe comparison, và các biểu đồ đa chiều (Browsers, Countries, v.v.).
*   **Custom Request Context:** Sử dụng hàm `getRequestContext` trong server-side để quản lý quyền truy cập (RBAC), kiểm tra project_id (x-pid) và các tham số timeframe (x-from, x-to) một cách tập trung.
*   **AI Agents & Function Calling:**
    *   Sử dụng kỹ thuật **Tool Use (Function Calling)** của OpenAI.
    *   Hệ thống định nghĩa các "Plugin" (Ví dụ: `VisitsPlugins.ts`, `DataPlugin.ts`). Khi người dùng hỏi AI, AI sẽ tự gọi các hàm này để lấy dữ liệu thực tế từ DB, sau đó mới tổng hợp thành câu trả lời.
*   **Dynamic Data Densification:** Kỹ thuật `$densify` trong MongoDB được sử dụng để lấp đầy các khoảng trống dữ liệu (ví dụ: ngày không có lượt truy cập nào) giúp biểu đồ hiển thị liên tục.
*   **Type-Safe Communication:** Tận dụng tRPC để Dashboard có thể gọi hàm từ dịch vụ Email/Payment giống như gọi hàm nội bộ, có nhắc lệnh (Intellisense) hoàn hảo.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng thu thập dữ liệu (Data Ingestion Flow)
1.  **Client:** Script `litlyx.js` trên website khách hàng gửi sự kiện (visit/event) về `Producer`.
2.  **Producer:**
    *   Kiểm tra IP có nằm trong Blacklist không.
    *   Kiểm tra Domain có trong Whitelist không.
    *   Tạo `sessionHash` và `flowHash`.
    *   Đẩy Object dữ liệu vào **Redis Stream (LITLYX_STREAM)**.
3.  **Consumer:**
    *   Luôn lắng nghe Redis Stream.
    *   Khi có dữ liệu, xác định chủ sở hữu dự án (Owner) và kiểm tra hạn mức (Limits).
    *   Sử dụng `ua-parser-js` để phân tích thiết bị/trình duyệt và `mmdb-lib` để định vị địa lý (IP Lookup).
    *   Lưu vào MongoDB (`VisitModel`, `EventModel`).
    *   Cập nhật bộ đếm tổng quát (`ProjectCountModel`).

#### B. Luồng phân tích & Hiển thị (Analytics Flow)
1.  **Dashboard:** Gửi request đến API với các Header `x-pid`, `x-from`, `x-to`.
2.  **API Server:**
    *   Kiểm tra cache trong Redis.
    *   Nếu không có cache, thực hiện Aggregation trên MongoDB.
    *   Trả kết quả về cho các component Vue xử lý hiển thị qua Chart.js.

#### C. Luồng AI Assistant
1.  **User:** Nhập câu hỏi "Traffic của tôi tuần này tăng hay giảm?".
2.  **AI Service:** Gửi lịch sử chat + câu hỏi sang OpenAI.
3.  **OpenAI:** Nhận diện nhu cầu cần dữ liệu, gọi "Tool" `getVisitsTimeline`.
4.  **Hệ thống:** Thực hiện truy vấn DB lấy số liệu thực, trả về cho AI.
5.  **AI Service:** Tổng hợp số liệu và trả về câu trả lời tự nhiên kèm nhận định chuyên sâu.

### Tổng kết
Litlyx là một hệ thống **Analytics tinh gọn nhưng mạnh mẽ**, ưu tiên tính module hóa và khả năng mở rộng. Việc sử dụng Redis Stream làm vùng đệm và tRPC để kết nối các service giúp hệ thống hoạt động mượt mà, trong khi việc tích hợp AI theo hướng "Agentic" (AI tự truy cập công cụ) tạo ra lợi thế cạnh tranh lớn so với các công cụ truyền thống.