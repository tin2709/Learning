Chào bạn, đây là bản phân tích chi tiết về dự án **Cially** dựa trên mã nguồn và tài liệu bạn cung cấp. Dự án này là một ví dụ điển hình về hệ thống **Full-stack Monorepo** hiện đại, kết hợp giữa thời gian thực (Real-time) và xử lý dữ liệu lớn (Data Processing).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án được tổ chức theo mô hình **Monorepo** sử dụng **Turborepo** và **pnpm workspaces**, giúp quản lý nhiều ứng dụng trong cùng một kho lưu trữ:

*   **Frontend (Web App):**
    *   **Next.js 15+ (App Router):** Sử dụng Server Components và Client Components.
    *   **Tailwind CSS 4.0:** Framework CSS mới nhất với hiệu suất cao.
    *   **Radix UI & Shadcn/UI:** Thư viện component giúp xây dựng giao diện nhanh và nhất quán.
    *   **Recharts:** Thư viện biểu đồ để trực quan hóa dữ liệu Discord.
*   **Discord Bot:**
    *   **Discord.js v14:** Thư viện tương tác với Discord API.
    *   **Express.js:** Chạy một API server ngay bên trong Bot để Web App có thể gọi ngược lại Bot.
*   **Backend & Database:**
    *   **Pocketbase (Go):** Một giải pháp Backend-as-a-Service (BaaS) nhỏ gọn dựa trên SQLite.
    *   **Go:** Dùng để tùy chỉnh Pocketbase (Custom Migrations & Hooks).
*   **DevOps:**
    *   **Docker & Docker Compose:** Đóng gói ứng dụng bot, web và pocketbase.
    *   **Biome:** Thay thế cho ESLint/Prettier để kiểm tra và định dạng mã nguồn cực nhanh.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Cially không chỉ đơn thuần là ghi log dữ liệu, kiến trúc của nó giải quyết bài toán **hiệu suất** và **giới hạn (Rate Limit)** của Discord:

*   **Kiến trúc 3 lớp tách biệt:**
    1.  **Collector (Bot):** Lắng nghe sự kiện (tin nhắn, người tham gia) và đẩy dữ liệu thô vào DB.
    2.  **Storage & Processor (Pocketbase):** Lưu trữ và chạy các tiến trình ngầm (Cron jobs) để tổng hợp dữ liệu (Aggregation).
    3.  **Presenter (Next.js):** Hiển thị dữ liệu đã được xử lý cho người dùng.
*   **Chiến lược "Raw to Stats":** Thay vì tính toán trực tiếp từ hàng triệu dòng tin nhắn mỗi khi người dùng mở biểu đồ, hệ thống lưu tin nhắn vào một bảng tạm, sau đó một tiến trình ngầm (Cron) sẽ tổng hợp chúng vào các bảng thống kê (`hourly_stats`, `user_stats`) và xóa tin nhắn thô đi để giữ database nhẹ.
*   **Hybrid Data Fetching:** Web App lấy dữ liệu số từ Pocketbase nhưng lấy dữ liệu tên/avatar (human-readable) từ Bot API. Điều này đảm bảo dữ liệu luôn cập nhật mà không cần lưu trữ quá nhiều thông tin định danh vào DB.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Batch Processing (Xử lý theo lô):**
    *   Trong file `pbAddNewData.js`, bot không lưu từng tin nhắn một mà sử dụng `pb.createBatch()` để gửi hàng nghìn bản ghi cùng lúc vào Pocketbase, giúp giảm thiểu overhead của HTTP request.
*   **Rate Limit Handling (Xử lý giới hạn tần suất):**
    *   Kỹ thuật **Exponential Backoff** được áp dụng trong `retryRequest.js`. Nếu database bị quá tải (lỗi 429), bot sẽ tự động chờ một khoảng thời gian tăng dần trước khi thử lại.
*   **Discord Scraping (Quét dữ liệu lịch sử):**
    *   Sử dụng kỹ thuật duyệt ngược (Cursor-based pagination) với `before: lastMessageId` để quét hàng triệu tin nhắn cũ mà không bị trùng lặp, kết hợp với `wait(waitFor)` để né tránh việc bị Discord ban do gọi API quá nhanh.
*   **Pocketbase Hooks (JSVM):**
    *   Sử dụng JavaScript bên trong Go (thông qua `pb_hooks`) để viết logic xử lý dữ liệu ngay tại tầng Database. File `messageOrganizer.pb.js` là trái tim của việc phân tích dữ liệu.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng ghi dữ liệu thời gian thực:
1.  Người dùng Discord gửi tin nhắn.
2.  `messageListener.js` của Bot bắt sự kiện -> Gửi POST request tới API nội bộ của Bot.
3.  Bot API xác thực và ghi vào bảng `messages` trong Pocketbase.

#### B. Luồng tổng hợp dữ liệu (The Background Engine):
1.  Mỗi phút, Pocketbase chạy Cron job `itemsOrganizer`.
2.  Nó đọc 10.000 bản ghi mới nhất từ bảng `messages`.
3.  Phân tích tin nhắn đó thuộc giờ nào, ngày nào, ai gửi, kênh nào.
4.  Cập nhật cộng dồn (+1) vào bảng `hourly_stats`, `user_stats`, `channel_stats`.
5.  Xóa bản ghi cũ trong `messages` để giải phóng dung lượng.

#### C. Luồng hiển thị (Dashboard):
1.  Người dùng truy cập Dashboard.
2.  Next.js gọi API route `/api/server/[id]/fetchActivityData`.
3.  Server-side của Next.js lấy các con số thống kê từ Pocketbase.
4.  Vì Pocketbase chỉ lưu ID (ví dụ: `channelID: "123..."`), Next.js gửi các ID này sang API của Bot (`/fetchID`).
5.  Bot dùng Cache/API của Discord để chuyển ID thành tên (ví dụ: `"General"`) và trả về cho Web.
6.  Next.js trộn dữ liệu và hiển thị lên biểu đồ Recharts.

---

### Tổng kết
**Cially** là một dự án có kiến trúc rất bài bản. Điểm sáng lớn nhất là việc **tách biệt nhiệm vụ**: Bot chỉ làm nhiệm vụ giao tiếp với Discord, Pocketbase làm nhiệm vụ tính toán dữ liệu, và Next.js chỉ làm nhiệm vụ hiển thị. Cách tiếp cận này giúp hệ thống có thể mở rộng (scale) tốt ngay cả khi server Discord có hàng trăm nghìn thành viên và lượng tin nhắn cực lớn.