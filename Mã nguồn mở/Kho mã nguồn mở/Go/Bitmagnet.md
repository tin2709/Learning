Dựa trên cấu trúc thư mục và mã nguồn của dự án **bitmagnet**, dưới đây là phân tích chi tiết về dự án theo các tiêu chí bạn yêu cầu:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng trên một ngăn xếp công nghệ mạnh mẽ, ưu tiên hiệu suất cao và khả năng tự vận hành (self-hosted):

*   **Ngôn ngữ lập trình chính:**
    *   **Go (Golang):** Lựa chọn hoàn hảo cho việc xử lý đồng thời (concurrency) cực lớn trong việc bò dữ liệu DHT (crawling) và quản lý hàng đợi công việc.
    *   **TypeScript/Angular:** Sử dụng cho Web UI, mang lại trải nghiệm ứng dụng đơn trang (SPA) mượt mờ và chuyên nghiệp với Angular Material.
*   **Cơ sở dữ liệu:**
    *   **PostgreSQL:** Không chỉ lưu trữ dữ liệu mà còn đóng vai trò là "trung tâm xử lý" với các tính năng nâng cao như:
        *   **Full-Text Search (FTS):** Tìm kiếm nội dung dựa trên `tsvector`.
        *   **JSONB:** Lưu trữ linh hoạt thông tin tập tin và thuộc tính mở rộng.
        *   **Hàng đợi (Queue):** Tận dụng PostgreSQL để làm hệ thống hàng đợi công việc (Job Queue) đáng tin cậy.
*   **Giao thức và API:**
    *   **G giao thức BitTorrent (DHT):** Thực thi các BEP (BitTorrent Extension Proposals) như BEP 5, 51, 33 để khám phá và lấy metadata.
    *   **GraphQL:** Sử dụng `gqlgen` để cung cấp API hiện đại cho giao diện người dùng.
    *   **Torznab:** Hỗ trợ chuẩn API dành cho các ứng dụng trong hệ sinh thái Servarr (Prowlarr, Radarr, Sonarr).
*   **Tích hợp bên thứ ba:**
    *   **TMDB (The Movie Database):** Nguồn dữ liệu chính để định danh và làm phong phú metadata cho phim và chương trình truyền hình.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án tuân thủ các nguyên tắc thiết kế hiện đại nhằm đảm bảo khả năng mở rộng và bảo trì:

*   **Dependency Injection (Uber fx):** Sử dụng thư viện `fx` để quản lý vòng đời ứng dụng và tiêm phụ thuộc. Điều này giúp các module (dht, classifier, queue, httpserver) hoạt động độc lập nhưng vẫn kết nối chặt chẽ.
*   **Hệ thống Worker-based:** Ứng dụng chạy theo mô hình đa tiến trình nền (background workers). Bạn có thể bật/tắt riêng lẻ các bộ phận như `http_server`, `queue_server`, hay `dht_crawler`.
*   **Kiến trúc hướng dữ liệu (Data-Driven):** Tư duy quản lý metadata tập trung. Mọi torrent sau khi tìm thấy đều đi qua một quy trình "lọc và làm giàu" (enrichment) trước khi được hiển thị.
*   **Tính module hóa cao:** Thư mục `internal/` được chia rõ ràng thành các package chức năng riêng biệt như `bloom` (lọc dữ liệu), `protocol` (xử lý mạng), `classifier` (phân loại), tránh tình trạng mã nguồn chồng chéo.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **DHT Crawling & Triage:**
    *   Sử dụng kỹ thuật lấy mẫu infohash (BEP 51) để phát hiện torrent mới mà không cần tracker trung tâm.
    *   Hệ thống "Triage" (phân loại nhanh) kiểm tra xem torrent đã tồn tại hoặc cần cập nhật (seeders/leechers) hay chưa để tối ưu hóa băng thông.
*   **Bộ phân loại nội dung bằng DSL (Domain Specific Language):**
    *   Sử dụng tệp YAML (`classifier.core.yml`) để định nghĩa logic phân loại. Cho phép người dùng tùy chỉnh cách nhận diện phim, nhạc, sách... dựa trên tên tệp và phần mở rộng mà không cần sửa code Go.
*   **Stable Bloom Filters:**
    *   Sử dụng bộ lọc Bloom để ghi nhớ hàng triệu infohash đã quét, giúp ngăn chặn việc xử lý lặp lại và tiết kiệm tài nguyên CPU/Disk.
*   **Budgeted Count (Đếm có ngân sách):**
    *   Kỹ thuật đặc biệt trong PostgreSQL để ước tính số lượng bản ghi lớn trong thời gian thực, giúp giao diện người dùng vẫn phản hồi nhanh ngay cả khi cơ sở dữ liệu có hàng chục triệu bản ghi.
*   **Banning & Filtering:**
    *   Cơ chế lọc tự động các nội dung rác hoặc nội dung độc hại (như CSAM) ngay từ giai đoạn thu thập metadata.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

Quy trình từ lúc phát hiện một Torrent đến khi xuất hiện trên UI:

1.  **Giai đoạn Khám phá (Discovery):** `dht_crawler` kết nối vào mạng lưới DHT toàn cầu, thu thập các Infohash đang được chia sẻ.
2.  **Giai đoạn Truy xuất Metadata:** Khi có infohash mới, hệ thống gửi yêu cầu qua giao thức dây (wire protocol) đến các peer để lấy metadata (tên tệp, kích thước, cấu trúc thư mục).
3.  **Hàng đợi Xử lý (Queueing):** Thông tin thô được đẩy vào `queue_jobs`.
4.  **Phân loại & Làm giàu (Classification & Enrichment):**
    *   Worker lấy job từ hàng đợi.
    *   `classifier` chạy logic DSL để đoán loại nội dung (phim, game, ...).
    *   Nếu là phim/TV, hệ thống gọi API TMDB để lấy poster, điểm số, thể loại chính xác.
5.  **Đánh chỉ mục (Indexing):** Dữ liệu hoàn chỉnh được lưu vào PostgreSQL, đồng thời cập nhật chỉ mục tìm kiếm văn bản (`tsvector`).
6.  **Cung cấp nội dung (Delivery):**
    *   Người dùng tìm kiếm trên Angular Web UI qua GraphQL API.
    *   Các ứng dụng như Radarr/Sonarr lấy dữ liệu qua endpoint Torznab.

### Tổng kết
`bitmagnet` là một dự án có độ hoàn thiện kỹ thuật rất cao, kết hợp giữa khả năng xử lý mạng ở mức thấp (BitTorrent protocol) và quản lý dữ liệu lớn ở mức cao (PostgreSQL/GraphQL), tạo ra một giải pháp tự lưu trữ (self-hosted) mạnh mẽ và độc lập.