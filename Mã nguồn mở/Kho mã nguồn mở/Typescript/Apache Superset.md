Dựa trên tài liệu và cấu trúc mã nguồn của **Apache Superset**, đây là phân tích chuyên sâu về hệ thống BI (Business Intelligence) mã nguồn mở này:

### 1. Công nghệ cốt lõi (Core Stack)

Superset là một ứng dụng monorepo kết hợp giữa hệ sinh thái Python (Backend) và React (Frontend):

*   **Backend (Python):**
    *   **Framework:** Flask và Flask-AppBuilder (FAB) cung cấp nền tảng quản lý người dùng, quyền (RBAC) và giao diện Admin.
    *   **ORM:** SQLAlchemy (Trụ cột để kết nối với gần như mọi cơ sở dữ liệu SQL).
    *   **Xử lý dữ liệu:** Pandas (Dùng để biến đổi kết quả truy vấn, tính toán post-processing).
    *   **Task Queue:** Celery + Redis/RabbitMQ (Xử lý các truy vấn dài, gửi báo cáo/cảnh báo qua email/Slack).
*   **Frontend (TypeScript/React):**
    *   **Thư viện UI:** Ant Design (đang được bao bọc lại bởi `@superset-ui/core`) kết hợp với Emotion để styling.
    *   **Quản lý State:** Redux và React Query.
    *   **Visualization:** Apache ECharts (chính), D3, deck.gl (bản đồ 3D).
*   **Hạ tầng:** Docker, Kubernetes (Helm), và WebSockets (Node.js service riêng để cập nhật trạng thái truy vấn thời gian thực).

### 2. Tư duy Kiến trúc (Architectural Mindsetmon)

Kiến trúc của Superset được thiết kế theo hướng **Cloud-Native** và **Extensible** (có khả năng mở rộng cực cao): Monolith nhưng có thể tách rời các worker.

*   **Lớp Semantic (Semantic Layer):** Superset không truy vấn trực tiếp bảng vật lý một cách tùy tiện. Nó định nghĩa một lớp trung gian (Datasets) nơi quản lý các cột ảo (Virtual Columns), các chỉ số (Metrics) và cấu hình quyền truy cập.
*   **Plugin-based Visualization:** Mỗi loại biểu đồ là một plugin riêng biệt. Kiến trúc này cho phép cộng đồng tự viết các loại chart mới mà không cần can thiệp vào core của Superset.
*   **Database Agnostic:** Thông qua SQLAlchemy và kiến trúc `db_engine_specs`, Superset trừu tượng hóa sự khác biệt giữa các phương ngữ SQL (ví dụ: cách xử lý thời gian trong BigQuery khác với PostgreSQL).
*   **Multi-tenant & Security:** Kiến trúc bảo mật dựa trên quyền hạn chi tiết đến từng cấp độ bảng và dòng (Row-Level Security - RLS).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Command Pattern (Backend):** Các logic nghiệp vụ phức tạp (như tạo Dashboard, import/export dữ liệu) được đóng gói trong thư mục `superset/commands/`. Điều này giúp tách biệt API, DAO (Data Access Object) và logic xử lý.
*   **DAO Pattern:** Sử dụng `daos/` để quản lý mọi thao tác với cơ sở dữ liệu metadata, giúp mã nguồn dễ kiểm thử (unit test) hơn.
*   **Jinja Templating trong SQL:** Cho phép lập trình viên viết SQL động. Người dùng có thể nhúng các biến như `{{ current_username() }}` trực tiếp vào truy vấn SQL.
*   **Functional Programming (Frontend):** Chuyển đổi mạnh mẽ từ Class Component sang Functional Component với React Hooks. Sử dụng kỹ thuật "Reactify" để bao bọc các thư viện chart không phải React (như D3) vào hệ sinh thái React.
*   **Monorepo Management:** Sử dụng Lerna để quản lý hàng chục package nhỏ trong `superset-frontend/packages/`.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một yêu cầu dữ liệu điển hình:

1.  **Client Request:** Người dùng tương tác với bộ lọc (Filter) trên Dashboard.
2.  **Query Object Construction:** Frontend xây dựng một `QueryObject` (chứa thông tin về metrics, group by, thời gian) thay vì gửi raw SQL để đảm bảo an toàn.
3.  **Backend Processing:**
    *   API nhận `QueryObject`.
    *   Lớp `db_engine_specs` dịch `QueryObject` thành câu lệnh SQL tương ứng với loại DB đang kết nối.
    *   Nếu là truy vấn dài, Celery Worker sẽ tiếp nhận và thực thi ngầm.
4.  **Execution:** Câu lệnh SQL chạy trên Database mục tiêu (Snowflake, ClickHouse, v.v.).
5.  **Data Transformation (Post-processing):** Kết quả trả về dưới dạng bảng (dataframe). Backend sử dụng Pandas để thực hiện các phép tính bổ sung (ví dụ: tính % tăng trưởng so với kỳ trước).
6.  **Visualization:** Dữ liệu JSON trả về Frontend, các Plugin Visualization nhận dữ liệu và vẽ lên màn hình thông qua ECharts hoặc D3.

### Đặc điểm mới đáng chú ý (MCP Service):
Trong mã nguồn có thư mục `superset/mcp_service/`. Đây là việc tích hợp **Model Context Protocol**, cho phép Superset kết nối với các AI Agent (như Claude Desktop). Kỹ thuật này giúp AI có thể "hiểu" schema dữ liệu và tự động tạo biểu đồ hoặc truy vấn thay cho người dùng.

### Tổng kết:
Superset là một hệ thống **truy vấn trừu tượng**. Nó không tập trung vào việc lưu trữ dữ liệu mà tập trung vào việc **quản lý vòng đời truy vấn** và **trực quan hóa kết quả** một cách bảo mật ở quy mô doanh nghiệp lớn.