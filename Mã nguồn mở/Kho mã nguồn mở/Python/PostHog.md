Dưới đây là bản phân tích chi tiết về dự án PostHog dưới dạng một file README tiếng Việt, tập trung vào các khía cạnh kỹ thuật chuyên sâu dành cho nhà phát triển.

---

# Phân tích Kỹ thuật Nền tảng PostHog

PostHog là một nền tảng "Product OS" mã nguồn mở cực kỳ phức tạp, kết hợp giữa khả năng phân tích dữ liệu quy mô lớn (Big Data) và các công cụ quản lý sản phẩm (Feature Flags, A/B Testing, Session Replay).

## 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án sử dụng mô hình **Monorepo** với sự kết hợp của nhiều ngôn ngữ để tối ưu hóa giữa tốc độ phát triển và hiệu suất thực thi:

### Backend (Đa ngôn ngữ)
*   **Python (Django):** Đóng vai trò là API trung tâm, quản lý logic nghiệp vụ, xác thực, và metadata. Django REST Framework (DRF) được dùng làm giao diện chính cho Frontend và CLI.
*   **Rust:** Được sử dụng cho các dịch vụ yêu cầu hiệu suất cực cao và độ trễ thấp, tiêu biểu là `capture` (nạp dữ liệu), `property-defs-rs`, và các dịch vụ xử lý plugin.
*   **Node.js (TypeScript):** Chạy `plugin-server` (hiện đang đổi tên thành `nodejs` trong repo), chịu trách nhiệm xử lý luồng dữ liệu (Data Pipelines), chạy các ứng dụng tùy chỉnh (HogFunctions) và xử lý sự kiện trong thời gian thực.
*   **Go:** Sử dụng cho dịch vụ `livestream`, giúp đẩy dữ liệu trực tiếp tới UI qua WebSocket.

### Cơ sở dữ liệu & Lưu trữ (Data Layer)
*   **ClickHouse:** "Trái tim" của PostHog. Đây là OLAP database giúp truy vấn hàng tỷ sự kiện với tốc độ mili giây.
*   **Kafka (Redpanda):** Hệ thống hàng đợi thông điệp (Message Queue) điều phối dữ liệu giữa các dịch vụ.
*   **PostgreSQL:** Lưu trữ metadata (user, team, configuration, dashboard definitions).
*   **Redis:** Caching, quản lý task (Celery), và lưu trữ trạng thái tạm thời cho Feature Flags.
*   **Object Storage (S3/MinIO):** Lưu trữ dữ liệu lớn như Session Recordings (Video quay lại màn hình người dùng).

### Frontend
*   **React + TypeScript:** Giao diện người dùng hiện đại.
*   **Kea:** Framework quản lý trạng thái (State Management) dựa trên Redux, giúp mã nguồn Frontend có tính cấu trúc rất cao.
*   **Lemon UI:** Thư viện component nội bộ của PostHog.

## 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của PostHog được xây dựng dựa trên nguyên tắc **"Data-First"** và **"Extreme Scalability"**:

1.  **Phân tách luồng Ghi và Đọc (CQRS):**
    *   Luồng ghi (Ingestion) cực nhanh thông qua Rust/Node.js đẩy vào Kafka.
    *   Luồng đọc (Query) phức tạp thông qua HogQL được thực thi trực tiếp trên ClickHouse.
2.  **HogQL (Hog Query Language):** PostHog không dùng SQL thuần mà tạo ra một lớp transpiler riêng. HogQL cho phép người dùng viết truy vấn giống SQL nhưng được tối ưu hóa tự động để chạy trên cấu trúc dữ liệu đặc thù của ClickHouse trong PostHog.
3.  **Kiến trúc Plugin/CDP:** Cho phép mở rộng tính năng mà không cần sửa core bằng cách chạy các đoạn mã JavaScript cô lập để biến đổi dữ liệu (Transformations) hoặc đẩy dữ liệu sang bên thứ 3 (Destinations).
4.  **Workflow Orchestration (Temporal):** Sử dụng Temporal để quản lý các tác vụ dài hơi, cần độ tin cậy cao như: đồng bộ dữ liệu kho (Data Warehouse sync), xuất file lớn (Exports), hoặc xóa dữ liệu hàng loạt.

## 3. Các kỹ thuật chính (Key Techniques)

*   **Autocapture & Biến đổi Schema:** Kỹ thuật thu thập mọi tương tác của người dùng mà không cần cấu trúc trước, sau đó sử dụng `Materialized Columns` trong ClickHouse để tăng tốc độ truy vấn các thuộc tính JSON phức tạp.
*   **Session Replay Compression:** Nén các sự kiện DOM và sự kiện mạng (network events) để tái tạo lại phiên làm việc của người dùng mà không gây tốn băng thông.
*   **Feature Flag Evaluation:** Thực thi logic cờ tính năng ngay tại biên (edge) hoặc local thông qua Redis để đảm bảo không làm chậm ứng dụng khách.
*   **Vector Search:** Tích hợp AI (HogAI) sử dụng các database vector (như pgvector hoặc chính ClickHouse) để phân tích hành trình người dùng bằng ngôn ngữ tự nhiên.

## 4. Tóm tắt luồng hoạt động (Project Workflow)

Luồng đi của một sự kiện (Event Lifecycle) trong PostHog:

1.  **Capture:** SDK (JS/Python/Mobile) gửi một sự kiện (ví dụ: `button_click`) đến dịch vụ `capture` (Rust).
2.  **Queue:** Sự kiện được kiểm tra API Key và đẩy vào **Kafka topic** `events_plugin_ingestion`.
3.  **Process:** Dịch vụ `nodejs` (Plugin Server) lấy dữ liệu từ Kafka:
    *   Định danh người dùng (Person identification).
    *   Gắn thêm thông tin địa lý (GeoIP).
    *   Chạy các Plugin transform (ví dụ: ẩn danh IP).
4.  **Storage:** Dữ liệu sau khi xử lý được đẩy vào Kafka một lần nữa trước khi được ghi vào **ClickHouse** (bảng `sharded_events`). Metadata được lưu vào **PostgreSQL**.
5.  **Analytics:** Khi người dùng mở Dashboard:
    *   Frontend gửi yêu cầu tới Django API.
    *   Django chuyển đổi yêu cầu thành **HogQL**.
    *   HogQL được dịch thành SQL của ClickHouse.
    *   ClickHouse trả về kết quả tổng hợp và hiển thị lên UI.
6.  **Replay:** Nếu tính năng ghi hình bật, các bản ghi DOM được nén và đẩy thẳng vào Object Storage, UI sẽ lấy dữ liệu này để phát lại clip.

---

**Kết luận:** PostHog là một ví dụ điển hình về việc xây dựng hệ thống phân tích quy mô lớn. Nó kết hợp sự linh hoạt của Django với sức mạnh thô của ClickHouse và Rust để tạo ra một trải nghiệm người dùng liền mạch từ dữ liệu thô đến thông tin chi tiết (insights).