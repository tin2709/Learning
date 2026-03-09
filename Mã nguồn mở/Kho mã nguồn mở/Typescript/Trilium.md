Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **TriliumNext (Trilium Notes)**:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án là một **Monorepo** (quản lý nhiều gói trong một kho lưu trữ) sử dụng `pnpm workspaces`.

*   **Ngôn ngữ chính:** **TypeScript** (chiếm đa số) đảm bảo an toàn về kiểu dữ liệu trên toàn bộ hệ thống từ Client đến Server.
*   **Frontend (Client):**
    *   **Preact:** Một thư viện nhẹ tương tự React để xây dựng UI hiện đại (đang dần thay thế các phần cũ).
    *   **jQuery:** Vẫn đóng vai trò quan trọng trong việc quản lý cây ghi chú (thông qua plugin **Fancytree**).
    *   **Vite:** Công cụ build cực nhanh cho môi trường phát triển.
    *   **Bootstrap & LightningCSS:** Dùng để xử lý giao diện và tối ưu hóa CSS.
*   **Backend (Server):**
    *   **Node.js & Express:** Framework xử lý các API REST và phục vụ web.
    *   **Better-SQLite3:** Thư viện SQLite cực nhanh cho Node.js, đóng vai trò là "trái tim" lưu trữ dữ liệu.
*   **Desktop:** **Electron**, cho phép chạy ứng dụng trên Windows, macOS và Linux với trải nghiệm bản địa.
*   **Trình soạn thảo (Editors):**
    *   **CKEditor 5:** Dùng cho ghi chú dạng văn bản giàu (Rich Text).
    *   **CodeMirror:** Dùng cho ghi chú dạng mã nguồn (Code).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Trilium xoay quanh khái niệm **"Local-first"** (ưu tiên dữ liệu cục bộ) và **Cấu trúc phân cấp vô hạn**.

*   **Mô hình dữ liệu Ghi chú - Nhánh (Note-Branch):**
    *   Một ghi chú không thuộc về một thư mục cố định. Thay vào đó, nó được kết nối thông qua các "nhánh" (Branch). Kỹ thuật này cho phép **Cloning**: Một ghi chú có thể xuất hiện ở nhiều vị trí khác nhau trong cây mà không bị nhân bản dữ liệu.
*   **Hệ thống Cache 3 lớp (The Triple-Cache System):** Đây là điểm đặc sắc nhất để đảm bảo hiệu suất cực cao:
    *   **Becca (Backend Cache):** Bản sao trong bộ nhớ của database SQLite trên server để truy vấn cực nhanh.
    *   **Froca (Frontend Cache):** Bản sao dữ liệu từ Server được đẩy xuống Client để UI có thể hiển thị ngay lập tức mà không cần đợi API.
    *   **Shaca (Share Cache):** Tối ưu hóa riêng cho các ghi chú được chia sẻ công khai qua web.
*   **Kiến trúc Widget (Widget-based UI):** Mọi thành phần giao diện (Thanh bên, trình soạn thảo, bản đồ...) đều được kế thừa từ `BasicWidget`. Cách tiếp cận này giúp dễ dàng mở rộng và tùy biến UI thông qua script của người dùng.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Hệ thống Scripting mạnh mẽ:** Trilium cho phép người dùng viết script JS chạy cả ở Frontend (browser) và Backend (Node.js). Điều này biến Trilium từ một app ghi chú thành một nền tảng quản lý tri thức có thể tự động hóa (ví dụ: tự tạo ghi chú hàng ngày, tracking cân nặng).
*   **Đồng bộ hóa (Synchronization):** Sử dụng WebSockets để đẩy thay đổi thời gian thực. Hệ thống theo dõi `entity_changes` để biết chính xác cái gì cần đồng bộ, giúp giảm thiểu băng thông.
*   **Thuộc tính Ghi chú (Attributes):** Sử dụng Label (nhãn) và Relation (quan hệ) để phân loại. Thuộc tính có thể kế thừa (Inheritance) từ ghi chú cha xuống ghi chú con.
*   **Bảo mật:** Sử dụng kỹ thuật mã hóa từng ghi chú (Per-note encryption). Có khái niệm "Protected Session" - người dùng phải nhập mật khẩu để xem các ghi chú nhạy cảm, và session này sẽ tự khóa sau một thời gian.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Người dùng thao tác:** Khi bạn viết một dòng văn bản, CKEditor sẽ xử lý.
2.  **Lưu trữ cục bộ:** Client gửi thay đổi qua API/WebSocket đến Server. Server lưu vào SQLite thông qua **Becca**.
3.  **Ghi nhận thay đổi:** Mỗi thay đổi tạo ra một bản ghi trong bảng `entity_changes`.
4.  **Phản hồi UI:** **Froca** trên Client cập nhật trạng thái ngay lập tức để người dùng không thấy độ trễ.
5.  **Đồng bộ hóa:** Nếu có nhiều thiết bị (Desktop, Server, Mobile), Server sẽ so sánh mã băm (hash) và các bản ghi thay đổi để đẩy dữ liệu mới nhất đến các thiết bị còn lại.
6.  **Mở rộng:** Nếu ghi chú là một script, hệ thống sẽ nạp nó vào một `ScriptContext` riêng biệt và thực thi các logic nghiệp vụ mà người dùng đã định nghĩa.

**Kết luận:** TriliumNext là một dự án có kỹ thuật rất cao, kết hợp giữa sự ổn định của các thư viện lâu đời (jQuery, SQLite) với tư duy kiến trúc hiện đại (Monorepo, TypeScript, Cache-first), phù hợp cho những người muốn xây dựng "bộ não thứ hai" thực thụ.