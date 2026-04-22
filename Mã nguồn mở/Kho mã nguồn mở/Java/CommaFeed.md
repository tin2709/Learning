Dựa trên toàn bộ mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **CommaFeed** — một hệ thống đọc RSS tự lưu trữ (self-hosted) hiện đại.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án có sự chuyển đổi mạnh mẽ từ kiến trúc truyền thống (Dropwizard) sang kiến trúc tối ưu hiệu suất (Quarkus).

*   **Backend (Java 25+):**
    *   **Quarkus:** Framework Java hướng Cloud Native, giúp khởi động cực nhanh và tiêu tốn cực ít RAM (dưới 50MB ở chế độ Native).
    *   **Hibernate / JPA:** Quản lý thực thể dữ liệu.
    *   **Liquibase:** Quản lý phiên bản Database (Migrations).
    *   **Rome:** Thư viện tiêu chuẩn để phân tích (parse) các định dạng RSS/Atom.
    *   **GraalVM:** Hỗ trợ biên dịch ra mã máy (Native Image) cho Windows/Linux/ARM64.
*   **Frontend (React & TypeScript):**
    *   **Vite:** Công cụ build frontend thế hệ mới (thay thế Webpack).
    *   **Redux Toolkit:** Quản lý trạng thái toàn cục (unread counts, settings, entries).
    *   **Mantine UI:** Thư viện component hiện đại, hỗ trợ Dark/Light mode và Responsive.
    *   **LinguiJS:** Quản lý đa ngôn ngữ (i18n).
*   **Database:** Hỗ trợ linh hoạt H2 (nhúng), PostgreSQL, MySQL và MariaDB.

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của CommaFeed tập trung vào **Hiệu suất (Performance)** và **Khả năng mở rộng (Scalability)** cho người dùng cá nhân hoặc cộng đồng nhỏ.

*   **Tách biệt hoàn toàn (Decoupled Frontend/Backend):** Backend chỉ đóng vai trò là một RESTful API. Frontend là một SPA (Single Page Application) giao tiếp qua JSON. Điều này cho phép dễ dàng xây dựng các client khác (Mobile app thông qua Fever API).
*   **Thiết kế hướng Native (Cloud Native & Native Executable):** Việc chọn Quarkus thay vì Spring Boot cho thấy tư duy tối ưu hóa tài nguyên server (phù hợp chạy trên VPS yếu hoặc Raspberry Pi).
*   **Cơ chế làm mới không đồng bộ (Asynchronous Refresh):** Việc cập nhật hàng triệu feed không chặn luồng chính. Hệ thống sử dụng một engine riêng (`FeedRefreshEngine`) với cơ chế xếp hàng (queue) và tính toán khoảng thời gian refresh dựa trên hoạt động của feed (Empirical Interval).
*   **Kiến trúc Database phẳng & Hiệu quả:** Các bảng được đánh Index kỹ lưỡng (đặc biệt là bảng `FeedEntryStatus`) để đảm bảo việc truy vấn hàng triệu bài viết vẫn mượt mà.

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **Backend:**
    *   **Lombok:** Giảm thiểu boilerplate code (Getter/Setter).
    *   **Thao tác dữ liệu Batch:** Khi dọn dẹp database (`DatabaseCleaningService`), hệ thống xóa theo lô (batch) để tránh treo DB.
    *   **Xử lý bảo mật:** Sử dụng cơ chế mã hóa Cookie phiên làm việc và hỗ trợ xác thực qua API Key cho các app bên thứ ba.
    *   **SSRF Mitigation:** Kỹ thuật chặn các địa chỉ IP nội bộ (`block-local-addresses`) để ngăn chặn tấn công giả mạo yêu cầu từ phía server.
*   **Frontend:**
    *   **Custom Hooks:** Tận dụng tối đa logic tái sử dụng (ví dụ: `useMousetrap` để quản lý phím tắt, `useWebSocket` để cập nhật thời gian thực).
    *   **Thunk Middleware:** Sử dụng `createAsyncThunk` trong Redux để xử lý các tác vụ bất đồng bộ như đánh dấu đã đọc bài viết hàng loạt.
    *   **Infinite Scrolling:** Kỹ thuật tải dữ liệu khi cuộn trang để tối ưu trải nghiệm người dùng trên danh sách dài.
    *   **Virtualization/Optimization:** Giảm số lượng render lại (re-render) bằng cách sử dụng `React.memo` và tối ưu hóa selectors trong Redux.

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Luồng Đăng ký Feed (Subscription Flow)
1.  Người dùng nhập URL (website hoặc RSS link).
2.  Backend sử dụng `FeedURLProvider` để tìm kiếm link RSS hợp lệ trong HTML (nếu người dùng nhập link website).
3.  `FeedFetcher` tải XML, `FeedParser` trích xuất thông tin (Title, Favicon).
4.  Lưu vào DB và trả về cho UI cập nhật cây danh mục (Tree).

#### B. Luồng Cập nhật Feed (Refresh Flow)
1.  `TaskScheduler` kích hoạt các job định kỳ.
2.  `FeedRefreshEngine` kiểm tra các feed đến hạn cập nhật.
3.  Hệ thống gửi yêu cầu HTTP với các header `If-Modified-Since` hoặc `ETag` để tiết kiệm băng thông (chỉ tải khi có bài mới).
4.  Nếu có bài mới, lưu vào bảng `FeedEntry`.
5.  **Real-time:** Thông báo qua **WebSocket** đến trình duyệt của người dùng để cập nhật số lượng bài chưa đọc (unread count) ngay lập tức mà không cần F5.

#### C. Luồng Xử lý Bài viết (Entry Processing)
1.  Người dùng click đọc bài -> Frontend gửi yêu cầu `markRequest`.
2.  Backend cập nhật trạng thái trong bảng `FeedEntryStatus`.
3.  Nếu người dùng thiết lập **Filtering Rules** (dùng CEL - Common Expression Language), bài viết có thể tự động bị đánh dấu là đã đọc nếu khớp từ khóa (ví dụ: quảng cáo).

### 5. Điểm nổi bật & Sáng tạo

1.  **Biên dịch Native:** Khả năng chạy như một file thực thi duy nhất (binary) không cần cài JRE trên máy khách.
2.  **Khả năng tùy biến cực cao:** Cho phép người dùng chèn **Custom CSS và JavaScript** trực tiếp từ giao diện cài đặt để thay đổi hoàn toàn diện mạo hoặc tính năng.
3.  **Fever API Compatibility:** Mặc dù là app hiện đại, nó vẫn hỗ trợ giao thức cũ (Fever) để tương thích với các app mobile nổi tiếng như Reeder, Unread.
4.  **Tự động hóa:** Tính năng tự động đánh dấu đã đọc sau X ngày hoặc dựa trên quy tắc lọc giúp người dùng quản lý tình trạng "ngập lụt" thông tin.

**Tổng kết:** CommaFeed là một ví dụ điển hình về việc áp dụng các công nghệ Java hiện đại nhất (Quarkus, Java 25) để hồi sinh một ứng dụng kiểu cũ (RSS Reader) với hiệu suất và trải nghiệm người dùng vượt trội.