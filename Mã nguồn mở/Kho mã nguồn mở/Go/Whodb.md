Dưới đây là bản phân tích chi tiết về dự án **WhoDB** dựa trên mã nguồn và tài liệu bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

WhoDB được xây dựng trên triết lý "nhẹ, nhanh và hiện đại", sử dụng các công nghệ mạnh mẽ nhất trong hệ sinh thái Go và TypeScript:

*   **Backend (GoLang):**
    *   **Go 1.25+:** Tận dụng hiệu suất cao, quản lý bộ nhớ tốt và khả năng biên dịch thành file thực thi duy nhất.
    *   **GORM:** Thư viện ORM chính để tương tác với các cơ sở dữ liệu SQL (Postgres, MySQL, SQLite, MariaDB).
    *   **GraphQL (gqlgen):** Toàn bộ API giao tiếp giữa Frontend và Backend đều qua GraphQL, giúp tối ưu hóa dữ liệu truyền tải.
    *   **Go-plugins:** Hệ thống plugin linh hoạt để hỗ trợ đa dạng loại DB (SQL, NoSQL như MongoDB, Key-Value như Redis, Search Engine như ElasticSearch).
*   **Frontend (React & TypeScript):**
    *   **Vite:** Công cụ build siêu nhanh thay cho CRA truyền thống.
    *   **Tailwind CSS:** Quản lý giao diện hiện đại và tùy biến cao.
    *   **Apollo Client:** Quản lý state và gọi API GraphQL.
    *   **Xyflow (React Flow):** Công nghệ cốt lõi để hiển thị sơ đồ quan hệ (Schema Topology) dưới dạng đồ thị tương tác.
*   **Giao diện dòng lệnh (CLI):**
    *   **Bubble Tea (Charmbracelet):** Framework TUI (Terminal UI) giúp tạo giao diện dòng lệnh đẹp mắt và tương tác như một ứng dụng GUI.
*   **Desktop App:**
    *   **Wails:** Sử dụng Go để điều khiển logic hệ thống và HTML/CSS/JS cho giao diện, tương tự Electron nhưng nhẹ hơn nhiều vì dùng Webview bản địa.
*   **AI Integration:**
    *   Hỗ trợ **Ollama** (chạy AI local), **OpenAI**, và **Anthropic** để chuyển đổi ngôn ngữ tự nhiên thành truy vấn SQL.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của WhoDB được thiết kế theo hướng **Modular (Mô-đun hóa)** và **Plugin-driven (Dựa trên Plugin)**:

*   **Plugin Architecture (Quan trọng nhất):** WhoDB không viết code cứng cho từng loại cơ sở dữ liệu. Thay vào đó, nó định nghĩa một interface chung (`PluginFunctions`). Mỗi loại DB là một plugin riêng biệt thực thi các hàm như `GetRows`, `ExecuteQuery`, `GetSchema`. Điều này cho phép mở rộng hỗ trợ các DB mới (như Clickhouse, Cassandra) mà không ảnh hưởng đến lõi hệ thống.
*   **Dual-Edition Strategy (CE/EE):** Sử dụng **Go Build Tags** (`//go:build ee`) để tách biệt bản Community (Mã nguồn mở) và Enterprise (Thương mại). Bản EE thường được đóng gói trong một submodule riêng (`ee/`), nếu không có nó, hệ thống sẽ tự động dùng các file `stub` (file rỗng) để đảm bảo bản CE vẫn build bình thường.
*   **GraphQL-First:** Tư duy "API là trên hết". Mọi tính năng mới phải được hiện thực hóa qua GraphQL trước khi đưa lên UI. Điều này giúp hệ thống đồng nhất, dễ test và hỗ trợ tốt cho CLI/Desktop.
*   **Lightweight & Zero-Dependency (Runtime):** Cố gắng giữ file thực thi dưới 50MB và tiêu tốn ít tài nguyên hơn 90% so với các công cụ truyền thống như DBeaver hay pgAdmin.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Table Virtualization (Ảo hóa bảng):** Sử dụng `react-window` để chỉ render những dòng dữ liệu đang hiển thị trên màn hình. Kỹ thuật này cho phép WhoDB duyệt qua hàng triệu dòng dữ liệu mà không làm treo trình duyệt.
*   **Parameterized Queries (Truy vấn tham số hóa):** Toàn bộ các thao tác dữ liệu từ UI đều được chuyển thành tham số để chống tấn công **SQL Injection**, một kỹ thuật bảo mật bắt buộc cho công cụ quản trị DB.
*   **Interactive Schema Topology:** Tự động phân tích Foreign Keys để vẽ sơ đồ quan hệ. Sử dụng các thuật toán bố cục đồ thị (layout algorithms) để sắp xếp các bảng một cách thông minh, giúp người dùng hiểu cấu trúc DB phức tạp ngay lập tức.
*   **MCP (Model Context Protocol):** Tích hợp giao thức mới nhất cho phép các tác nhân AI (như Claude Code, Cursor) có thể "nói chuyện" trực tiếp với cơ sở dữ liệu thông qua WhoDB CLI.
*   **Mock Data Generation:** Tích hợp thư viện `gofakeit` để sinh dữ liệu mẫu trực tiếp vào bảng, hỗ trợ đắc lực cho việc phát triển và kiểm thử (QA).

---

### 4. Tóm tắt luồng hoạt động (Activity Flow Summary)

1.  **Khởi tạo & Kết nối:**
    *   Người dùng nhập thông tin kết nối qua Web UI, Desktop hoặc CLI.
    *   Backend nhận yêu cầu, xác định loại DB và gọi Plugin tương ứng.
    *   Hệ thống kiểm tra kết nối và lưu trữ session (WhoDB ưu tiên không lưu mật khẩu vĩnh viễn để bảo mật, trừ khi người dùng cấu hình Profile).
2.  **Khám phá (Discovery):**
    *   Sau khi kết nối, Backend chạy các truy vấn metadata để lấy danh sách Schema, Table, Columns và quan hệ Foreign Keys.
    *   Dữ liệu này được đẩy về Frontend để dựng lên sơ đồ cây (Sidebar) và sơ đồ đồ thị (Schema View).
3.  **Thao tác dữ liệu (Data Manipulation):**
    *   Người dùng xem bảng: Frontend gọi query GraphQL `GetStorageUnitRows` -> Plugin thực hiện `SELECT` với `LIMIT/OFFSET` -> Trả về grid dữ liệu.
    *   Người dùng sửa ô (Inline editing): Frontend gửi mutation GraphQL -> Backend tạo câu lệnh `UPDATE` dựa trên Primary Key -> Thực thi và trả về kết quả thành công.
4.  **Truy vấn AI (AI-Powered Query):**
    *   Người dùng nhập: "Lấy 10 khách hàng mua nhiều hàng nhất tháng trước".
    *   WhoDB gửi yêu cầu kèm theo cấu trúc schema (chỉ tên bảng/cột, không gửi dữ liệu) đến LLM (Ollama/OpenAI).
    *   LLM trả về câu lệnh SQL. WhoDB hiển thị SQL này cho người dùng kiểm tra trước khi thực thi.
5.  **Xuất dữ liệu (Export):**
    *   Hệ thống sử dụng các stream writer để chuyển đổi kết quả truy vấn trực tiếp thành file CSV/Excel/JSON mà không nạp toàn bộ vào RAM, giúp xuất các file dữ liệu lớn cực nhanh.

**Kết luận:** WhoDB không chỉ là một trình quản lý DB, mà là một **Data Explorer thế hệ mới** kết hợp giữa sức mạnh hệ thống của Go và khả năng tương tác linh hoạt của AI và Đồ thị.