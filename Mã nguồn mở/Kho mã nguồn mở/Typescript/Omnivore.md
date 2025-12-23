Dựa trên mã nguồn và cấu trúc thư mục của dự án **Omnivore**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và các kỹ thuật cốt lõi:

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án là một hệ sinh thái đầy đủ (Full-stack Ecosystem) chạy theo mô hình **Monorepo**:

*   **Ngôn ngữ chính:** TypeScript (chiếm đa số ở Backend/Frontend), Kotlin (Android), Swift (iOS), Python (ML/Digest Score).
*   **Backend:** Node.js với Apollo GraphQL Server. Đây là "trái tim" điều phối dữ liệu qua GraphQL API.
*   **Frontend Web:** Next.js, Stitches (styling), Radix UI (component), và SWR để xử lý data fetching.
*   **Cơ sở dữ liệu:**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ chính. Sử dụng `pgvector` cho các tính năng liên quan đến AI/Embedding.
    *   **Redis:** Đóng vai trò làm hàng đợi tin nhắn (Queue) thông qua Bullmq và caching.
*   **Xử lý nội dung (Content Processing):**
    *   **Puppeteer:** Chạy Chromium không giao diện để render các trang web SPA phức tạp trước khi lưu.
    *   **Mozilla Readability:** Trích xuất nội dung chính từ HTML (loại bỏ quảng cáo, menu).
*   **Mobile:** 
    *   **Android:** Kotlin, Jetpack Compose, Room DB (Offline support), Hilt (DI), Apollo Kotlin.
    *   **iOS:** Swift, SwiftUI, Swift GraphQL.
*   **AI/ML:** Tích hợp OpenAI và AWS Bedrock để tóm tắt bài viết và chấm điểm tin tức (Digest score).

---

### 2. Tư duy kiến trúc (Architectural Mindset)

Omnivore được thiết kế với tư duy **"Offline-first"** và **"Microservices"**:

*   **Kiến trúc Monorepo:** Sử dụng Lerna để quản lý nhiều gói (packages) trong một kho chứa duy nhất. Điều này giúp chia sẻ logic (như `content-handler` hay `readability`) giữa API server và các dịch vụ worker.
*   **Kiến trúc hướng hàng đợi (Queue-driven Architecture):** Các tác vụ nặng như tải nội dung, xử lý PDF, gửi email không chạy trực tiếp trên API server mà được đẩy vào Redis queue. Các gói như `content-fetch`, `pdf-handler`, `thumbnail-handler` đóng vai trò là các worker tiêu thụ hàng đợi này.
*   **Kiến trúc Mobile (Android Example):** 
    *   Áp dụng **Clean Architecture** phân lớp rõ rệt: `core/data` (Repository), `core/database` (Entity/DAO), `feature/*` (UI/ViewModel).
    *   Sử dụng **Single Source of Truth**: Mọi dữ liệu hiển thị trên UI đều lấy từ Local Database (Room), việc đồng hành với server diễn ra ngầm thông qua các Sync Worker.
*   **Kiến trúc Plugin:** Hỗ trợ tốt cho Obsidian và Logseq thông qua việc mở các API endpoint chuyên biệt.

---

### 3. Các kỹ thuật chính (Main Techniques)

*   **Đồng bộ hóa dữ liệu (Advanced Sync Logic):** 
    *   Sử dụng trường `serverSyncStatus` (NEEDS_CREATION, NEEDS_UPDATE, IS_SYNCED...) để đánh dấu trạng thái đồng bộ của từng bản ghi (highlights, articles) trên mobile. 
    *   Kỹ thuật "Merge Highlights": Xử lý xung đột khi người dùng highlight cùng một đoạn văn bản trên các thiết bị khác nhau.
*   **Content Extraction (Trích xuất đa nguồn):**
    *   Xây dựng các `handler` riêng biệt cho từng loại nguồn (Substack, Twitter, YouTube, PDF).
    *   Kỹ thuật render nội dung qua WebView bằng cách nhúng các tệp JS/CSS tùy chỉnh (như `mathJaxConfiguration.js`) để hỗ trợ công thức toán học và chế độ đọc tối ưu.
*   **Vector Search & AI:**
    *   Sử dụng `pgvector` trong PostgreSQL để thực hiện tìm kiếm ngữ nghĩa (Semantic Search) thay vì chỉ tìm kiếm từ khóa thông thường.
*   **Self-hosting & Dockerization:**
    *   Dự án cung cấp hệ thống `docker-compose` hoàn chỉnh bao gồm cả Mail server (để nhận bản tin newsletter), Image proxy (để tối ưu ảnh) và API.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

#### Luồng Lưu một bài viết (Saving Process):
1.  **Client:** Người dùng lưu link qua Extension hoặc App. Link được gửi đến API Server.
2.  **API Server:** Ghi nhận yêu cầu vào Database, tạo một Task và đẩy vào hàng đợi (Redis).
3.  **Content Fetcher (Worker):** Nhận Task, dùng Puppeteer để tải trang.
4.  **Readability:** Trích xuất text, ảnh, metadata từ HTML thô.
5.  **Storage:** Nội dung sạch được lưu vào DB (hoặc S3/GCS nếu là PDF), ảnh được Image Proxy xử lý.
6.  **Notification:** API server thông báo cho Client qua GraphQL Subscription hoặc Webhook.

#### Luồng Đọc và Highlight (Reading & Highlighting):
1.  **App:** Tải nội dung đã trích xuất về Local DB để đọc offline.
2.  **User:** Tạo một highlight. App lưu ngay vào Room DB với trạng thái `NEEDS_CREATION`.
3.  **Sync Manager:** Một tiến trình chạy ngầm phát hiện thay đổi local, gọi API GraphQL để cập nhật lên Server.
4.  **Server:** Cập nhật DB và đồng bộ trạng thái về `IS_SYNCED` cho thiết bị.

### Kết luận
Dự án Omnivore là một mẫu hình tiêu biểu cho việc kết hợp giữa **TypeScript backend** mạnh mẽ và **Native Mobile apps**. Điểm mạnh nhất của dự án nằm ở khả năng **xử lý nội dung thô biến thành trải nghiệm đọc tinh gọn** thông qua một pipeline xử lý dữ liệu phức tạp nhưng có độ trễ thấp. Với việc chuyển sang mô hình hoàn toàn **Self-hosted**, kiến trúc của nó hiện tại tập trung tối đa vào tính module hóa để người dùng dễ dàng triển khai trên hạ tầng riêng.