Dưới đây là phân tích chi tiết về dự án **jsdeob-workbench**, một công cụ trực quan mạnh mẽ dành cho việc đảo ngược kỹ thuật (reverse engineering) và giải mã (deobfuscation) mã nguồn JavaScript.

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án được xây dựng trên một ngăn xếp công nghệ chuyên biệt cho việc phân tích cú pháp và thao tác mã nguồn:

*   **Babel (Hệ sinh thái chủ đạo):** Dự án sử dụng toàn bộ "bộ sậu" của Babel để xử lý mã nguồn:
    *   `@babel/parser`: Chuyển đổi mã JS thô thành cây cú pháp trừu tượng (AST).
    *   `@babel/traverse`: Duyệt cây AST và thực hiện các thay đổi tại từng nút (node).
    *   `@babel/types`: Thư viện tiện ích để kiểm tra, xây dựng và thay đổi các loại nút AST.
    *   `@babel/generator`: Chuyển đổi ngược AST đã được chỉnh sửa về mã JS có thể đọc được.
*   **Monaco Editor:** Sử dụng bộ soạn thảo mã nguồn của VS Code để cung cấp trải nghiệm nhập/xuất code chuyên nghiệp (highlighting, gợi ý).
*   **Web Workers:** Sử dụng luồng chạy ngầm để thực thi các tác vụ tính toán AST nặng nề, giúp giao diện người dùng (UI) luôn mượt mà và không bị treo khi xử lý các file obfuscate lớn.
*   **Express & Node.js:** Cung cấp backend để quản lý các plugin, dự án và chạy các transform phía máy chủ (khi cần dung lượng bộ nhớ lớn cho stack size).
*   **pkg & esbuild:** Sử dụng `esbuild` để đóng gói Babel cho trình duyệt và `pkg` để đóng gói toàn bộ ứng dụng thành tệp thực thi (.exe) độc lập.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của dự án được thiết kế theo mô hình **Pipeline (Đường ống)** và **Plugin-based (Dựa trên Plugin)**:

*   **Mô hình "Recipe Chain":** Lấy cảm hứng từ công cụ *CyberChef*, dự án cho phép người dùng xếp chồng các bộ lọc (transforms) lên nhau. Kết quả của bộ lọc trước là đầu vào của bộ lọc sau. Điều này giúp chia nhỏ các kỹ thuật giải mã phức tạp thành các bước nhỏ, dễ quản lý.
*   **Phân tách xử lý (Hybrid Processing):** 
    *   Các tác vụ nhanh được xử lý qua **Web Worker** ở Client để giảm tải cho server.
    *   Các tác vụ cực nặng (như giải mã mã nguồn lồng nhau sâu - JSFuck) được gửi về **Server Node.js**, nơi có thể cấu hình `--stack-size` lớn hơn giới hạn của trình duyệt.
*   **Tính trừu tượng hóa Plugin:** Dự án coi mỗi đoạn mã Babel là một "Plugin". Plugin có thể được nạp từ file hệ thống (`plugins/` folder) hoặc viết trực tiếp trên giao diện. Kiến trúc này cho phép mở rộng không giới hạn mà không cần can thiệp vào mã nguồn lõi.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Thao tác AST chuyên sâu:** Thay vì sử dụng Regex (vốn rất dễ lỗi với JS), dự án sử dụng các kỹ thuật xử lý cây cú pháp như:
    *   **Constant Folding:** Sử dụng `path.evaluate()` để tính toán các biểu thức hằng số ngay lập tức (ví dụ: `1 + 2` thành `3`).
    *   **Scope Analysis:** Phân tích phạm vi biến để phát hiện biến không sử dụng hoặc đổi tên biến một cách an toàn.
*   **Visitor Pattern:** Đây là kỹ thuật lập trình chính trong việc duyệt AST. Mỗi transform định nghĩa một đối tượng "Visitor" để lắng nghe các sự kiện khi đi qua các loại nút cụ thể (như `StringLiteral`, `MemberExpression`).
*   **Sandbox Execution (DANGER Zone):** Dự án cung cấp hàm `run()` để thực thi mã JS giải mã. Kỹ thuật này thường dùng để chạy các hàm giải mã chuỗi của chính mã nguồn bị obfuscate để lấy kết quả thật.
*   **Tối ưu hóa Parse/Generate:** Thay vì Parse và Generate lại mã sau mỗi bước transform (gây tốn hiệu năng), dự án cố gắng giữ trạng thái AST xuyên suốt chuỗi transform và chỉ Generate mã cuối cùng khi kết thúc chuỗi.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý dữ liệu diễn ra tuần tự như sau:

1.  **Input:** Người dùng dán mã JavaScript bị obfuscate vào Monaco Editor bên trái.
2.  **Parsing:** Hệ thống sử dụng Babel Parser để tạo ra một **AST duy nhất**.
3.  **Transformation Pipeline:**
    *   Hệ thống duyệt qua danh sách "Recipe" do người dùng thiết lập.
    *   Với mỗi bước, nó nạp mã script của Plugin, tạo một môi trường thực thi ảo (thông qua `Function` constructor) và truyền AST cùng các helper (`ast`, `traverse`, `t`) vào.
    *   AST được thay đổi trực tiếp (In-place modification). Các thay đổi được ghi lại vào `stats`.
4.  **Refining (Tùy chọn):** Người dùng có thể sử dụng **AST Viewer** để kiểm tra cấu trúc cây tại các nút nghi vấn hoặc sử dụng **Scope Analysis** để xem các liên kết biến.
5.  **Generation:** Babel Generator nhận AST cuối cùng, thực hiện làm đẹp mã (`Beautify`) và xuất kết quả ra Monaco Editor bên phải.
6.  **Summary:** Hệ thống hiển thị biểu đồ và thống kê (thời gian chạy, kích thước code giảm bao nhiêu %) để người dùng đánh giá mức độ hiệu quả của "Recipe".

### Tổng kết
**jsdeob-workbench** là một công cụ trung gian hoàn hảo giữa việc viết script thủ công và các công cụ giải mã tự động. Nó cung cấp cho chuyên gia phân tích mã độc một môi trường "thí nghiệm" để thử nghiệm nhanh các mẫu giải mã trên cây AST trước khi áp dụng chính thức.