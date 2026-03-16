Dựa trên các tệp tin mã nguồn và tài liệu kiến trúc bạn cung cấp, dưới đây là phân tích chi tiết về dự án **World Monitor**. Đây là một hệ thống giám sát thông tin tình báo nguồn mở (OSINT) cực kỳ phức tạp và hiện đại.

### 1. Công nghệ cốt lõi (Core Tech Stack)

Hệ thống được xây dựng theo mô hình đa nền tảng (Web, Desktop) và đa môi trường (Edge, Cloud, Local).

*   **Frontend (Browser):**
    *   **Vanilla TypeScript & Preact:** Sử dụng TypeScript thuần và Preact (một phiên bản siêu nhẹ của React) để tối ưu hiệu năng cho các dashboard nhiều dữ liệu.
    *   **Dual Map Engine:** Kết hợp **globe.gl (Three.js)** cho hiển thị quả địa cầu 3D và **deck.gl (MapLibre GL)** cho các lớp dữ liệu WebGL phẳng 2D.
    *   **Data Visualization:** Sử dụng **D3.js** cho các biểu đồ và sparklines.
*   **Desktop App:**
    *   **Tauri v2 (Rust):** Đóng gói ứng dụng web thành ứng dụng desktop native, sử dụng Rust để quản lý hệ thống bảo mật (keychain) và vòng đời ứng dụng.
    *   **Node.js Sidecar:** Một quy trình Node.js chạy song song với ứng dụng Tauri để xử lý các logic backend cục bộ và proxy API.
*   **Backend & API Layer:**
    *   **Vercel Edge Functions:** Hơn 60 endpoint API chạy trên môi trường Edge để giảm độ trễ tối đa.
    *   **Sebuf (RPC Framework):** Một framework tùy chỉnh dựa trên **Protocol Buffers (Protobuf)** để định nghĩa các hợp đồng API nghiêm ngặt và tự động tạo mã (codegen).
    *   **Railway Relay:** Sử dụng để duy trì các kết nối WebSocket (cho luồng AIS tàu biển) và các tác vụ chạy ngầm (cron/seeding).
*   **AI/ML:**
    *   **Local AI (Ollama):** Hỗ trợ chạy các mô hình ngôn ngữ lớn (LLM) trực tiếp trên máy người dùng.
    *   **Transformers.js:** Chạy các mô hình học máy (như MiniLM cho nhúng văn bản, phân tích cảm xúc) ngay trong trình duyệt qua Web Workers.
*   **Data & Caching:**
    *   **Upstash Redis:** Lưu trữ cache tập trung, chống "cache stampede" và quản lý Rate Limiting.
    *   **Cloudflare R2:** Lưu trữ bản đồ (PMTiles) và các tệp dữ liệu tĩnh lớn.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của World Monitor tập trung vào **Khả năng phục hồi (Resilience)** và **Quyền riêng tư (Privacy)**:

*   **Mô hình "Gold Standard" Data:** Dữ liệu được các script "seeder" thu thập từ hơn 30 nguồn (GDELT, ACLED, Yahoo Finance...) sau đó chuẩn hóa và đẩy vào Redis. Các API endpoint chỉ việc đọc từ Redis, giúp hệ thống không bị sập khi các API nguồn bị lỗi hoặc giới hạn lượt gọi.
*   **Kiến trúc Phân cấp Cache (4 lớp):**
    1.  Seed dữ liệu vào Redis.
    2.  Cache trong bộ nhớ (In-memory) tại Edge.
    3.  Redis tập trung.
    4.  Gọi API gốc nếu tất cả các lớp trên không có dữ liệu.
*   **Hệ thống Variant (Biến thể):** Chỉ từ một mã nguồn duy nhất, hệ thống có thể tạo ra 5 trang web khác nhau (World, Tech, Finance, Commodity, Happy) bằng cách thay đổi cấu hình các Panel và Map Layer thông qua biến môi trường.
*   **Tư duy Offline-First cho Desktop:** Ứng dụng desktop được thiết kế để có thể hoạt động hoàn toàn cục bộ nếu người dùng cấu hình Ollama, giúp dữ liệu tình báo không bao giờ rời khỏi máy tính cá nhân.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Sebuf RPC:** Thay vì dùng REST truyền thống, dự án định nghĩa Service trong file `.proto`. Kỹ thuật này đảm bảo Frontend và Backend luôn đồng bộ về kiểu dữ liệu (Type-safe).
*   **Bootstrap Hydration:** Thay vì gửi hàng trăm yêu cầu API khi mở app, SPA gọi một endpoint `/api/bootstrap` để lấy "gói" dữ liệu khởi động cho tất cả các panel cùng một lúc.
*   **Circuit Breakers:** Sử dụng mẫu thiết kế ngắt mạch (`circuit-breaker.ts`) để ngăn chặn việc app bị treo khi một dịch vụ dữ liệu (ví dụ: dữ liệu động đất) gặp sự cố kéo dài.
*   **Jaccard Similarity:** Sử dụng thuật toán này trong Web Worker để nhóm các tin tức tương đồng từ hàng trăm nguồn tin khác nhau, tránh trùng lặp nội dung.
*   **Solar Terminator Overlay:** Kỹ thuật tính toán và hiển thị vùng sáng/tối (ngày/đêm) trên bản đồ theo thời gian thực.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Thu thập dữ liệu (Seeding):** Các script chạy trên Railway định kỳ lấy dữ liệu thô từ các vệ tinh, sàn chứng khoán, và nguồn tin quân sự. Dữ liệu này được làm sạch, gán tọa độ địa lý và lưu vào Upstash Redis kèm metadata về độ tươi (freshness).
2.  **Khởi tạo App:** Khi người dùng truy cập, App xác định "Variant" (ví dụ: Finance). Nó tải các tệp ngôn ngữ (i18n) và gọi `/api/bootstrap` để lấy dữ liệu cache từ Redis.
3.  **Hiển thị:** 
    *   Bản đồ 3D/2D được vẽ. 
    *   Hơn 80 loại "Panel" (thành phần giao diện) được khởi tạo. 
    *   Dữ liệu được "truyền" vào các Panel thông qua `AppContext`.
4.  **Phân tích thời gian thực:** 
    *   Web Workers bắt đầu chạy ngầm để phân tích xu hướng từ khóa (Keyword Spikes).
    *   Hệ thống Correlation Engine tìm kiếm mối liên hệ giữa các sự kiện (ví dụ: một vụ nổ quân sự có thể liên quan đến sự sụt giảm giá dầu).
5.  **Tương tác:** Khi người dùng click vào một quốc gia, hệ thống thực hiện "Reverse Geocode" để xác định vị trí, sau đó truy vấn "Country Intelligence Index" để hiển thị tóm tắt tình hình an ninh, kinh tế của quốc gia đó bằng AI.

**Kết luận:** World Monitor không chỉ là một trang web bản đồ, mà là một **hệ điều hành tình báo nguồn mở**. Sự kết hợp giữa Protobuf cho dữ liệu, WebGL cho hiển thị và Local LLM cho phân tích biến nó thành một công cụ có năng lực tiệm cận các phần mềm chuyên dụng như Palantir.