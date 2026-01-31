Dựa trên cấu trúc thư mục và nội dung các file bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Streamystats** dưới góc nhìn kỹ thuật:

---

### 1. Công Nghệ Cốt Lõi (Core Technology Analysis)

Streamystats được xây dựng trên một Stack hiện đại, ưu tiên hiệu suất cao và tính toàn vẹn dữ liệu:

*   **Runtime & Package Manager:** **Bun (v1.3+)**. Đây là lựa chọn chiến lược giúp tăng tốc độ thực thi JavaScript/TypeScript, đặc biệt là khả năng biên dịch (compile) thành file thực thi đơn lẻ (binary) trong Docker.
*   **Backend Framework:** **Hono**. Một framework cực nhẹ chạy trên Bun, được sử dụng cho `job-server` để xử lý các logic nặng nề và API hiệu suất cao.
*   **Frontend:** **Next.js (App Router)** với **TypeScript**. Sử dụng các thư viện UI hiện đại như **Tailwind CSS**, **shadcn/ui** và **Lucide React**.
*   **Database & ORM:**
    *   **PostgreSQL** kết hợp với extension **pgvector (Vectorchord)**: Cho phép lưu trữ và truy vấn vector (embeddings) để phục vụ tính năng tìm kiếm ngữ nghĩa và gợi ý AI.
    *   **Drizzle ORM:** Cung cấp khả năng truy vấn Type-safe và quản lý migrations mạnh mẽ.
*   **Job Queue:** **pg-boss**. Sử dụng chính database PostgreSQL làm hàng đợi công việc, giúp quản lý các tác vụ chạy ngầm (sync dữ liệu, tạo embeddings) một cách tin cậy.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án áp dụng mô hình **Monorepo** với sự phân tách trách nhiệm (Separation of Concerns) rõ ràng:

*   **Kiến trúc Đa Dịch vụ (Multi-service):**
    *   `apps/nextjs-app`: Đảm nhận vai trò Web UI và các API phục vụ trực tiếp người dùng.
    *   `apps/job-server`: Một dịch vụ riêng biệt chuyên xử lý các tác vụ tốn tài nguyên và chạy định kỳ (Cron jobs).
    *   `packages/database`: Một gói dùng chung chứa Schema và logic kết nối DB, đảm bảo tính nhất quán dữ liệu giữa hai ứng dụng trên.
*   **Tư duy "AI-Native":** Khác với các app thống kê thông thường, Streamystats tích hợp AI ngay từ tầng dữ liệu (vector database) và cung cấp các "Tools" để AI có thể truy cập trực tiếp vào thông tin thư viện của người dùng.
*   **Khả năng mở rộng (Scalability):** Dự án hỗ trợ cả chế độ All-in-One (AIO) cho người dùng cá nhân dễ cài đặt và chế độ Docker riêng biệt cho từng service để tối ưu hóa tài nguyên.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Vector Similarity Search (Tìm kiếm tương đồng):** Sử dụng khoảng cách Cosine trên `pgvector` để so sánh sở thích xem phim của người dùng với nội dung trong thư viện, từ đó đưa ra các đề xuất "tương đồng về nội dung" thay vì chỉ dựa trên thể loại.
*   **Geolocation & Anomaly Detection:**
    *   Sử dụng `geoip-lite` để xác định vị trí từ IP trong Activity Log.
    *   Thuật toán **Impossible Travel**: Tính toán tốc độ di chuyển giữa hai lần đăng nhập để phát hiện truy cập bất thường (ví dụ: đăng nhập ở Mỹ rồi 10 phút sau ở Việt Nam).
*   **Function Calling (AI Tools):** Hệ thống định nghĩa 13 công cụ chuyên biệt (Search, Stats, Recommendations) cho phép Chatbot AI thực hiện các hành động thực tế trên dữ liệu người dùng thay vì chỉ trả lời văn bản suông.
*   **SSE (Server-Sent Events):** Sử dụng để đẩy dữ liệu thời gian thực từ `job-server` về trình duyệt (như tiến độ sync dữ liệu hoặc cảnh báo bảo mật) mà không cần refresh trang.
*   **Complex Exclusion Logic:** Kỹ thuật lọc dữ liệu thống kê phức tạp (loại trừ user hoặc thư viện nhất định) được triển khai nhất quán ở tầng DB query để đảm bảo tính chính xác của báo cáo.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng dữ liệu của Streamystats diễn ra theo chu trình khép kín:

1.  **Kết nối & Đồng bộ hóa ban đầu:**
    *   Người dùng thiết lập kết nối tới Jellyfin qua API Key.
    *   `job-server` kích hoạt chuỗi Jobs: Sync Users -> Libraries -> Items (Phim/Series) -> People (Diễn viên/Đạo diễn).
2.  **Thu thập dữ liệu thời gian thực:**
    *   `session-poller` liên tục quét Jellyfin API (mỗi 5s) để bắt các phiên phát trực tuyến đang diễn ra.
    *   Activity Log được sync định kỳ để ghi nhận lịch sử đăng nhập/phát sóng.
3.  **Làm giàu dữ liệu (Data Enrichment):**
    *   **Geolocation Job:** Chuyển IP thành tọa độ địa lý.
    *   **Fingerprinting Job:** Phân tích thiết bị và vị trí thường xuyên của user để tạo "dấu vân tay" bảo mật.
    *   **Embedding Job:** Gửi mô tả phim/metadata tới AI Provider (OpenAI/Ollama) để nhận về vector đặc trưng và lưu vào Vector DB.
4.  **Tiêu thụ dữ liệu (Consumption):**
    *   **Dashboard:** Next.js render các biểu đồ thống kê từ dữ liệu đã xử lý.
    *   **AI Recommendations:** Khi user yêu cầu gợi ý, hệ thống lấy lịch sử xem gần đây -> tạo vector sở thích -> truy vấn vector DB tìm các phim tương đồng nhất.
    *   **Security Monitor:** Nếu phát hiện vị trí mới hoặc thiết bị lạ, hệ thống bắn event SSE để hiển thị cảnh báo ngay lập tức trên UI.

Tóm lại, **Streamystats** là một hệ thống phân tích dữ liệu chuyên sâu, tận dụng tối đa sức mạnh của **Vector Database** và **AI Agents** để biến các con số thống kê khô khan thành trải nghiệm tương tác thông minh cho người dùng Jellyfin.