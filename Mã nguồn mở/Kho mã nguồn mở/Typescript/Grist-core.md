Dựa trên mã nguồn và cấu trúc thư mục của **Grist-core**, đây là một phân tích chuyên sâu về một trong những nền tảng "Spreadsheet-Database" (Bảng tính lai Cơ sở dữ liệu) mã nguồn mở tiên tiến nhất hiện nay.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Grist không chọn những giải pháp phổ thông mà kết hợp các công nghệ đặc thù để giải quyết bài toán hiệu năng và bảo mật:

*   **Ngôn ngữ:** 
    *   **TypeScript (81.5%):** Chiếm đa số, dùng cho cả Frontend và Backend (Node.js). Đảm bảo tính chặt chẽ của kiểu dữ liệu (Type-safety) cho một hệ thống quản lý logic phức tạp.
    *   **Python (11.4%):** Đây là "linh hồn" của công cụ tính toán (Data Engine). Grist sử dụng Python để xử lý các công thức (formulas) thay vì JavaScript, cho phép người dùng sử dụng toàn bộ thư viện chuẩn của Python.
*   **Cơ sở dữ liệu:**
    *   **SQLite:** Mỗi tài liệu Grist thực chất là một file SQLite. Điều này giúp dữ liệu cực kỳ cơ động (portable), hỗ trợ SQL query trực tiếp và đảm bảo tính toàn vẹn dữ liệu.
    *   **TypeORM:** Dùng để quản lý các dữ liệu hệ thống (User, Org, Workspace) trong các DB như PostgreSQL hoặc chính SQLite.
*   **Formula Engine & Sandboxing (Công nghệ then chốt):**
    *   **Pyodide (WASM):** Chạy Python trực tiếp trong trình duyệt bằng WebAssembly.
    *   **gVisor (Google):** Dùng trên Docker/Linux để tạo môi trường sandbox cô lập hoàn toàn việc thực thi mã Python của người dùng, đảm bảo an toàn cho máy chủ.
*   **Frontend:**
    *   **GrainJS:** Một thư viện reactive (phản xạ) do chính đội ngũ Grist phát triển, tập trung vào việc quản lý DOM hiệu quả và khả năng "disposal" (giải phóng bộ nhớ) cực mạnh.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Grist được xây dựng quanh khái niệm **"Relational Spreadsheet"**:

*   **Hybrid Data Model:** Khác với Excel (vốn coi mỗi ô là một thực thể độc lập), Grist coi **Cột (Column)** là thực thể định nghĩa kiểu dữ liệu (như Database). Điều này cho phép tạo ra các liên kết (References) giữa các bảng một cách chặt chẽ.
*   **Dependency Graph (Đồ thị phụ thuộc):** Hệ thống duy trì một đồ thị các ô và công thức. Khi một ô thay đổi, Grist chỉ tính toán lại tối thiểu các ô bị ảnh hưởng, tương tự như cách các build system (như Bazel hay Gradle) hoạt động.
*   **Separation of Concerns (Tách biệt thực thi):**
    *   **Home Server:** Quản lý người dùng, phân quyền, danh sách tài liệu.
    *   **Doc Worker:** Chịu trách nhiệm mở file, tính toán công thức và quản lý transaction của từng tài liệu cụ thể.
*   **Plug-and-play UI:** Sử dụng hệ thống **Custom Widgets** (thư mục `plugins/core`), cho phép nhúng các thành phần giao diện bên ngoài vào bảng tính thông qua IFrame API.

---

### 3. Kỹ thuật lập trình (Programming Techniques)

Nhìn vào code, chúng ta thấy những kỹ thuật rất cao cấp:

*   **Observables & Data Binding:** Sử dụng `GrainJS` để tạo ra luồng dữ liệu hai chiều. Khi dữ liệu trong SQLite thay đổi, UI tự động cập nhật mà không cần load lại trang.
*   **JS-Python Bridge:** Grist xây dựng một giao thức truyền tin (RPC) cực kỳ tinh vi giữa Node.js và môi trường Sandbox Python. Dữ liệu được "marshal" (đóng gói) và gửi qua pipe để tính toán.
*   **Migrations (Gen-server/migration):** Hệ thống quản lý phiên bản schema DB rất chặt chẽ, cho phép nâng cấp tài liệu cũ lên các phiên bản Grist mới hơn mà không mất dữ liệu.
*   **Intensive Testing:** Thư mục `test/` cực kỳ đồ sộ với các bài test tích hợp (nbrowser) sử dụng Mocha/Selenium, kiểm thử từ logic tính toán đến hành vi UI của người dùng.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Hãy lấy ví dụ luồng khi một người dùng nhập một công thức `SUM(Table1.Amount)`:

1.  **Giao diện (Frontend):** Người dùng nhập công thức vào ô. `AceEditor` (trong `app/client/components`) cung cấp gợi ý mã (autocompletion).
2.  **Yêu cầu (Client -> Server):** Một Action được gửi lên Node.js server thông qua WebSocket.
3.  **Xử lý (Data Engine):**
    *   Node.js chuyển yêu cầu vào Sandbox (Python).
    *   Python parse công thức, xác định các cột phụ thuộc.
    *   Data Engine truy vấn dữ liệu từ SQLite.
    *   Thực thi mã Python để lấy kết quả.
4.  **Phản hồi (Server -> Client):** Kết quả được lưu vào SQLite, đồng thời một thông điệp "DocAction" được phát (broadcast) tới tất cả người dùng đang mở tài liệu đó.
5.  **Cập nhật (UI):** Frontend nhận Action, `GrainJS` thực hiện cập nhật DOM cục bộ tại ô đó.

---

### Tổng kết

**Grist-core** là một ví dụ mẫu mực về việc:
1.  **Dùng đúng công cụ cho đúng việc:** Python cho toán học/logic, TS cho hệ thống, SQLite cho lưu trữ.
2.  **Bảo mật là ưu tiên:** Đầu tư cực lớn vào Sandboxing (gVisor/WASM) để cho phép người dùng chạy code tùy ý.
3.  **Tự chủ công nghệ:** Phát triển `GrainJS` để kiểm soát hoàn toàn vòng đời của UI, tránh phụ thuộc vào các Framework béo phì như React nếu không cần thiết.

Đây là một dự án tuyệt vời để học về **System Design** (hệ thống phân tán), **Security Sandboxing** và **Performance Optimization** trong các ứng dụng web phức tạp.