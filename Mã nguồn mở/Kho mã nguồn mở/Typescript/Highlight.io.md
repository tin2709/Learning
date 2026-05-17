Dựa trên cấu trúc thư mục và nội dung tệp tin của repository **Highlight.io**, đây là một phân tích chuyên sâu về hệ thống quan sát (observability) full-stack này:

### 1. Công nghệ Cốt lõi (Core Technology Stack)

Highlight được xây dựng trên một kiến trúc hiện đại, tập trung vào việc xử lý dữ liệu lớn và thời gian thực:

*   **Backend (Go):** Sử dụng ngôn ngữ Go cho hiệu suất cao, xử lý song song tốt. Framework chính là **Chi** (HTTP router) và **GQLGen** để xây dựng API GraphQL.
*   **Frontend (React/TypeScript):** Ứng dụng dashboard được xây dựng bằng **Vite**, **Apollo Client** (để giao tiếp GraphQL), và **Tailwind CSS** kết hợp với **Vanilla Extract** (CSS-in-JS có tính typesafe).
*   **Lưu trữ (Hybrid Database):**
    *   **PostgreSQL (GORM):** Lưu trữ dữ liệu quan hệ, thông tin người dùng, cài đặt project và metadata.
    *   **ClickHouse:** Đây là "trái tim" của hệ thống analytics. Nó lưu trữ logs, traces, và session events (dạng cột - columnar) để phục vụ các truy vấn tìm kiếm tốc độ cao trên hàng tỷ bản ghi.
    *   **Redis:** Dùng để caching và quản lý trạng thái phiên làm việc (session).
*   **Hàng đợi & Xử lý (Kafka):** Sử dụng **Apache Kafka** để làm bộ đệm cho luồng dữ liệu khổng lồ đẩy về từ các SDK trước khi ghi vào ClickHouse.
*   **Giao thức dữ liệu:** Dựa trên tiêu chuẩn **OpenTelemetry (OTel)**, giúp Highlight dễ dàng tích hợp với các hệ thống khác và thu thập dữ liệu chuẩn hóa.

### 2. Tư duy Kiến trúc (Architectural Philosophy)

*   **Dual GraphQL API:** 
    *   *Public Graph:* Chuyên biệt cho việc nhận dữ liệu từ các SDK (ingestion). Nó được tối ưu để chịu tải cực lớn.
    *   *Private Graph:* Dành cho ứng dụng dashboard frontend, tập trung vào việc truy vấn, lọc dữ liệu và quản trị.
*   **Kiến trúc Producer-Consumer:** Tách biệt hoàn toàn việc nhận dữ liệu (API) và ghi dữ liệu (Worker) thông qua Kafka. Điều này đảm bảo nếu database quá tải, dữ liệu vẫn không bị mất mà chỉ tạm thời nằm trong hàng đợi.
*   **Monorepo Management:** Sử dụng **Turborepo** và **Yarn Workspaces** để quản lý hàng chục package (SDK, UI, AI, Render) trong cùng một nơi, giúp việc phát triển đồng bộ giữa frontend và backend trở nên dễ dàng hơn.

### 3. Kỹ thuật Lập trình Đặc sắc (Notable Programming Techniques)

*   **Session Replay (rrweb):** Highlight sử dụng và đóng góp cho `rrweb` để ghi lại các thay đổi của DOM dưới dạng snapshot và các biến động (mutations). Thay vì quay video màn hình, họ lưu lại các event JSON, giúp dung lượng lưu trữ cực nhỏ và có thể tìm kiếm nội dung bên trong session.
*   **ANTLR for Search Grammar:** Trong thư mục `antlr/`, họ định nghĩa một bộ ngữ pháp (`SearchGrammar.g4`) để parse các chuỗi tìm kiếm phức tạp của người dùng (ví dụ: `level:error service:backend ...`) thành các câu lệnh SQL cho ClickHouse.
*   **AI-Powered Grouping:** Có sự xuất hiện của các package `ai` và `predictions` (Python/Go), sử dụng OpenAI để tự động gom nhóm các lỗi tương tự và đề xuất giải pháp sửa lỗi cho lập trình viên.
*   **Code Generation:** Tận dụng tối đa việc tạo mã tự động (code-gen) từ GraphQL schema cho cả Go (backend) và TypeScript (frontend), đảm bảo tính nhất quán và giảm thiểu lỗi typo.

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng Ingestion (Thu thập dữ liệu):
1.  **Client/Server SDK:** Ghi lại lỗi, log hoặc vết (trace) từ ứng dụng của người dùng.
2.  **Public Graph (API):** Nhận payload qua HTTPS/GraphQL/OTLP.
3.  **Kafka Queue:** API đẩy message vào Kafka (topic: `default`, `traces`, `logs`).
4.  **Worker:** Các worker Go consume từ Kafka, thực hiện các logic như: khử trùng lặp (de-duplication), làm giàu dữ liệu (enrichment) và lưu vào ClickHouse/S3.

#### B. Luồng Visualization (Hiển thị):
1.  **Frontend Dashboard:** Người dùng thực hiện thao tác lọc hoặc xem một session.
2.  **Private Graph (API):** Nhận yêu cầu, truy vấn đồng thời vào Postgres (lấy metadata) và ClickHouse (lấy dữ liệu thô).
3.  **Data Rendering:** Kết quả được trả về React để hiển thị biểu đồ hoặc tái hiện lại phiên làm việc thông qua trình phát Replayer.

### 5. Khả năng Mở rộng (Scalability)
Nhờ việc sử dụng **ClickHouse** và **Kafka**, hệ thống có khả năng scale ngang (horizontal scaling). Các thành phần như `public-worker`, `private-graph` có thể chạy độc lập trong các container Docker (thấy rõ trong thư mục `deploy/`), cho phép Highlight xử lý từ các dự án nhỏ (Hobby) đến các doanh nghiệp lớn (Enterprise) với lưu lượng hàng triệu event mỗi giây.

**Tóm lại:** Highlight.io là một ví dụ điển hình về việc áp dụng triệt để kiến trúc hướng sự kiện (event-driven) và cơ sở dữ liệu phân tích hiện đại để giải quyết bài toán quan sát ứng dụng một cách toàn diện.