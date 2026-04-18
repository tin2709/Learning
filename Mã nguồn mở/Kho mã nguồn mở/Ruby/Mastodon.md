Dựa trên các tệp tin và cấu trúc thư mục mà bạn cung cấp, Mastodon là một hệ thống mạng xã hội phi tập trung (decentralized) cực kỳ phức tạp và được tối ưu hóa cao. Dưới đây là phân tích chi tiết:

### 1. Công nghệ cốt lõi (Core Technologies)
Mastodon sử dụng một "Stack" công nghệ đa dạng để giải quyết các bài toán về hiệu suất và tính mở rộng:

*   **Backend chính:** **Ruby on Rails (8.1.0)**. Đây là trái tim xử lý logic nghiệp vụ, REST API và quản lý cơ sở dữ liệu.
*   **Real-time Streaming:** **Node.js**. Mastodon tách riêng phần xử lý WebSocket (thông báo thời gian thực) sang một service chạy Node.js (thư mục `/streaming`) để tránh làm nghẽn server Rails khi có hàng ngàn kết nối duy trì đồng thời.
*   **Cơ sở dữ liệu:**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (Người dùng, bài viết, quan hệ theo dõi).
    *   **Redis:** Đóng vai trò cực kỳ quan trọng trong việc lưu trữ Cache, quản lý hàng đợi công việc (Sidekiq) và Home Feed của người dùng.
    *   **Elasticsearch:** Sử dụng thông qua gem `chewy` (thư mục `app/chewy`) để tìm kiếm toàn văn (full-text search) cho bài viết, tài khoản và hashtag.
*   **Frontend:** **React.js + Redux**. Giao diện người dùng là một ứng dụng Single Page Application (SPA) hiện đại, quản lý trạng thái phức tạp qua Redux.
*   **Giao thức phi tập trung:** **ActivityPub**. Đây là "ngôn ngữ" giúp các server Mastodon khác nhau có thể nói chuyện và trao đổi dữ liệu với nhau (thư mục `app/controllers/activitypub`).
*   **Asset Pipeline:** **Vite**. Chuyển từ Webpack sang Vite để tăng tốc độ phát triển và build ứng dụng frontend.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Mastodon được thiết kế xung quanh khái niệm **"Decentralization" (Phi tập trung)** và **"Asynchronous" (Bất đồng bộ)**:

*   **Federation (Liên bang hóa):** Mastodon không coi dữ liệu chỉ nằm trong server nội bộ. Kiến trúc được thiết kế để liên tục "đẩy" (Push) dữ liệu ra bên ngoài và "kéo" (Pull) dữ liệu từ các server khác về thông qua các `ActivityPub Controllers` và `Serializers`.
*   **Kiến trúc dựa trên Worker (Worker-Centric):** Gần như mọi hành động tốn thời gian (gửi email, đẩy bài viết tới hàng ngàn người theo dõi, tải ảnh từ server khác) đều được đẩy vào hàng đợi **Sidekiq** (`app/workers`). Điều này giúp API luôn phản hồi nhanh (Latency thấp).
*   **Service-Oriented Logic:** Thay vì để logic trong Model hay Controller, Mastodon sử dụng rất nhiều **Service Objects** (`app/services`). Ví dụ: `PostStatusService`, `FollowService`, `ResolveAccountService`. Điều này giúp mã nguồn dễ kiểm thử và bảo trì.
*   **Cơ chế lưu trữ "Blob" linh hoạt:** Hỗ trợ lưu trữ file trên hệ thống cục bộ hoặc các dịch vụ đám mây (S3, Azure, OpenStack) thông qua Paperclip.

### 3. Kỹ thuật Lập trình Chính (Key Programming Techniques)

*   **Xử lý định dạng JSON-LD:** Mastodon sử dụng các `Serializers` chuyên dụng (`app/serializers/activitypub`) để biến đổi các đối tượng Ruby thành chuẩn JSON-LD (ActivityStreams) mà mạng lưới Fediverse có thể hiểu.
*   **Signature Verification (Xác thực chữ ký):** Để đảm bảo tính bảo mật trong mạng lưới phi tập trung, Mastodon sử dụng `SignatureVerification` concern. Mọi yêu cầu từ server khác gửi đến đều phải được xác thực chữ ký HTTP để đảm bảo danh tính.
*   **Paginable & Searchable:** Sử dụng các Concern như `Paginable` để quản lý phân trang dữ liệu lớn và `Searchable` để tích hợp mượt mà với Elasticsearch qua Chewy.
*   **Rate Limiting & Anti-Spam:** Tích hợp `Rack::Attack` và các validator tùy chỉnh (`app/validators`) để ngăn chặn tấn công và spam, điều rất quan trọng với một hệ thống mở.
*   **Internationalization (i18n):** Hệ thống hỗ trợ đa ngôn ngữ cực kỳ mạnh mẽ (thư mục `config/locales`), cho phép cộng đồng đóng góp dịch thuật dễ dàng qua Crowdin.

### 4. Luồng hoạt động của Hệ thống (System Workflow)

#### Luồng 1: Người dùng nội bộ đăng một bài viết (Toot)
1.  **Client (React):** Gửi yêu cầu POST đến `api/v1/statuses`.
2.  **Controller:** Tiếp nhận, xác thực qua `doorkeeper` (OAuth2).
3.  **Service:** `PostStatusService` được gọi để lưu bài viết vào Postgres.
4.  **Fan-out (Lan tỏa):** Đây là bước quan trọng nhất. Một worker `DistributionWorker` sẽ được kích hoạt để:
    *   Đẩy bài viết vào Home Feed của những người theo dõi cùng server (lưu vào Redis).
    *   Gửi thông báo qua WebSocket (Node.js Streaming API).
    *   Với những người theo dõi ở server khác, `ActivityPub::DeliveryWorker` sẽ gửi dữ liệu bài viết qua HTTP POST tới Inbox của các server đó.

#### Luồng 2: Tiếp nhận dữ liệu từ server khác (Federation)
1.  **Inbox:** Một server khác gửi yêu cầu đến `/api/v1/inboxes`.
2.  **Authentication:** `InboxesController` kiểm tra chữ ký HTTP (HTTP Signature).
3.  **Background Processing:** Yêu cầu được đưa vào `ActivityPub::ProcessingWorker` để không làm treo kết nối HTTP.
4.  **Parsing:** `ActivityPub::Parser` phân tích JSON nhận được. Nếu là một bài đăng mới, nó sẽ gọi `ProcessStatusService` để lưu vào DB địa phương (dưới dạng bản sao cache) và hiển thị cho người dùng trong server.

### Đánh giá tổng quát:
Kiến trúc của Mastodon là một ví dụ mẫu mực về cách xây dựng hệ thống mạng xã hội quy mô lớn bằng Ruby on Rails. Nó giải quyết bài toán phi tập trung bằng cách kết hợp chặt chẽ giữa các tiêu chuẩn web mở (ActivityPub) và một hệ thống xử lý tác vụ nền (Background Jobs) cực kỳ mạnh mẽ. Việc tách biệt phần Streaming sang Node.js cho thấy tư duy thực dụng trong việc chọn công cụ phù hợp cho từng bài toán hiệu suất.