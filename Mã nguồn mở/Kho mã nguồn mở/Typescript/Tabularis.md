Dưới đây là phân tích chi tiết về dự án **Tabularis** — một công cụ quản trị cơ sở dữ liệu (Database Client) hiện đại, mã nguồn mở, dựa trên nội dung mã nguồn và tài liệu bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Stack)

Dự án sử dụng mô hình **Hybrid Desktop App** cực kỳ hiện đại, tối ưu cho cả hiệu suất backend và trải nghiệm người dùng frontend:

*   **Framework chính:** **Tauri v2**. Đây là yếu tố quan trọng giúp ứng dụng nhẹ hơn nhiều so với Electron vì sử dụng WebView của hệ điều hành và backend bằng Rust.
*   **Backend (Rust):**
    *   **SQLx:** Bộ công cụ database mạnh mẽ để giao tiếp với PostgreSQL, MySQL và SQLite.
    *   **JSON-RPC 2.0:** Giao thức nền tảng để giao tiếp với các Plugin bên ngoài qua `stdin/stdout`.
    *   **Tokio:** Runtime bất đồng bộ để xử lý nhiều kết nối database cùng lúc.
*   **Frontend (React 19 & TypeScript):**
    *   **Tailwind CSS v4:** Framework CSS mới nhất để xử lý giao diện.
    *   **Monaco Editor:** Bộ soạn thảo code mạnh mẽ (giống VS Code) để viết SQL.
    *   **React Flow (xyflow):** Dùng để xây dựng "Visual Query Builder" (truy vấn dạng kéo thả) và "ER Diagram".
    *   **TanStack Table & Virtual:** Xử lý hiển thị bảng dữ liệu lớn (Data Grid) một cách mượt mà.
    *   **Recharts:** Hiển thị biểu đồ trong SQL Notebooks.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Tabularis tập trung vào **Tính mở (Extensibility)** và **Hiệu suất (Performance)**:

*   **Kiến trúc Bridge (Cầu nối):** Tauri tạo ra một lớp giao tiếp an toàn giữa UI (JavaScript) và System (Rust). Các tác vụ nặng như tính toán schema, SSH Tunneling, và kết nối DB được đẩy hoàn toàn xuống Rust.
*   **Hệ thống Plugin "Ngôn ngữ trung lập":** Khác với các app chỉ cho phép viết plugin bằng JS, Tabularis dùng **JSON-RPC qua stdin/stdout**. Điều này cho phép lập trình viên viết Driver/Plugin bằng *bất kỳ ngôn ngữ nào* (Go, Python, Rust, Node.js) miễn là nó xuất ra được file thực thi.
*   **Slot-based UI Extension:** Sử dụng tư duy "Slots" (các vị trí chờ sẵn). Plugin không chỉ xử lý dữ liệu mà còn có thể "chèn" các component React vào các vị trí như Toolbar, Sidebar hay Context Menu của app chính thông qua `@tabularis/plugin-api`.
*   **Tư duy AI-Native:** Tabularis không chỉ tích hợp AI vào chatbot mà còn biến mình thành một **MCP Server** (Model Context Protocol). Điều này giúp các trợ lý AI (như Claude, Cursor) có thể trực tiếp "đọc" và "hiểu" cấu trúc database của người dùng để thực hiện truy vấn.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **SQL Notebooks:** Kỹ thuật trộn lẫn Markdown và SQL (tương tự Jupyter Notebook). Đặc biệt là khả năng **tham chiếu biến chéo cell** (`{{cellName.columnName}}`), yêu cầu hệ thống phải xây dựng được đồ thị phụ thuộc (dependency graph) giữa các câu lệnh SQL.
*   **Visual EXPLAIN:** Chuyển đổi kết quả thô của lệnh `EXPLAIN` (vốn rất khó đọc) thành dạng đồ thị (Node Graph) trực quan, giúp lập trình viên tìm ra các "điểm nghẽn" (expensive scans, joins) trong truy vấn.
*   **Virtual Scrolling:** Kỹ thuật render dữ liệu cực lớn trong Data Grid. Chỉ những dòng đang hiển thị trên màn hình mới được vẽ vào DOM, giúp app không bị lag khi mở bảng có hàng triệu bản ghi.
*   **SSH Tunneling tự động:** Tích hợp sẵn khả năng tạo đường truyền bảo mật đến database nằm trong mạng nội bộ mà không cần cài thêm công cụ ngoài.
*   **Zero-bundle Plugin API:** Sử dụng kỹ thuật *Externalize*. Các plugin UI khi build sẽ không kèm theo React hay API của app, mà sẽ sử dụng các bản được "tiêm" (inject) sẵn từ ứng dụng chủ để giữ dung lượng plugin cực nhỏ.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi động:**
    *   Backend (Rust) quét các thư mục Plugin để đọc `manifest.json`.
    *   Đăng ký các Driver (Native và Plugin) vào hệ thống.
    *   Frontend load các UI Extension từ plugin và render vào các "Slots".
2.  **Kết nối Database:**
    *   Người dùng nhập thông tin $\rightarrow$ Rust tạo Pool kết nối (qua SQLx).
    *   Hệ thống chạy các truy vấn Metadata song song (Parallel fetching) để lấy danh sách bảng, cột, khóa ngoại... và cập nhật sidebar.
3.  **Thực thi truy vấn (Query Flow):**
    *   Người dùng viết SQL trong Monaco Editor.
    *   Frontend gửi SQL qua IPC (Inter-Process Communication) tới Backend.
    *   Backend thực thi, parse kết quả thành dạng JSON chuẩn hóa và trả về.
    *   Frontend sử dụng cơ chế phân trang (Pagination) và hiển thị lên Data Grid.
4.  **Tương tác AI:**
    *   Người dùng hỏi AI $\rightarrow$ Tabularis cung cấp context (Schema) cho LLM.
    *   AI sinh SQL $\rightarrow$ Tabularis thực thi và trả kết quả hiển thị dạng bảng hoặc biểu đồ ngay trong Notebook.

---

**Kết luận:** Tabularis là một dự án có độ phức tạp kỹ thuật cao, kết hợp nhuần nhuyễn giữa sự an toàn của Rust và sự linh hoạt của hệ sinh thái React. Nó định nghĩa lại cách một Database Client hoạt động trong kỷ nguyên AI.