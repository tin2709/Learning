Dựa trên nội dung kho lưu trữ **Onlook**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này bằng tiếng Việt:

---

### 1. Công nghệ cốt lõi (Tech Stack)

Onlook sử dụng một tập hợp các công nghệ hiện đại, tập trung vào hiệu suất và khả năng mở rộng của một công cụ lập trình trực quan:

*   **Frontend & UI:**
    *   **Next.js (App Router):** Framework chính cho toàn bộ ứng dụng web.
    *   **TailwindCSS:** Hệ thống styling chủ đạo, cũng là đối tượng chính mà trình chỉnh sửa trực quan của Onlook tác động vào.
    *   **tRPC & Zod:** Đảm bảo kiểu dữ liệu an toàn (type-safe) giữa client và server.
    *   **MobX:** Quản lý trạng thái (state management) cho các store phức tạp của trình soạn thảo (Editor Engine).
*   **Backend & Data:**
    *   **Supabase:** Quản lý Authentication, Database (PostgreSQL) và Real-time.
    *   **Drizzle ORM:** Quản lý cấu trúc dữ liệu và truy vấn DB một cách linh hoạt.
*   **AI & Khả năng xử lý:**
    *   **Vercel AI SDK:** Client để tương tác với các LLM.
    *   **OpenRouter:** Cổng kết nối với nhiều mô hình ngôn ngữ lớn khác nhau.
    *   **Morph Fast Apply & Relace:** Các model chuyên biệt để áp dụng nhanh các thay đổi code từ AI vào dự án.
*   **Runtime & Tooling:**
    *   **Bun:** Được dùng làm package manager và runtime chính thay cho npm/yarn nhờ tốc độ cao và hỗ trợ monorepo tốt.
    *   **CodeSandbox SDK:** Tạo môi trường sandbox (container) để chạy code trực tiếp trên trình duyệt.
    *   **Docker:** Quản lý container cho việc self-hosting.

---

### 2. Kỹ thuật và Tư duy kiến trúc

Kiến trúc của Onlook được thiết kế để giải quyết bài toán khó nhất: **Làm sao để đồng bộ hóa 100% giữa giao diện kéo thả (Visual) và mã nguồn (Code).**

*   **Kiến trúc "Code-as-Source-of-Truth":** Mọi thay đổi trên giao diện đều được chuyển đổi thành các thao tác chỉnh sửa trực tiếp vào file code. Onlook không lưu trữ một định dạng file riêng mà đọc/ghi trực tiếp vào JSX/TSX.
*   **Instrumentation (Định vị mã nguồn):** Khi code chạy trong sandbox, Onlook thực hiện "đo đạc" (instrument) để gắn nhãn các phần tử DOM với vị trí chính xác của chúng trong file mã nguồn. Điều này cho phép tính năng "Chuột phải vào element -> Mở đúng dòng code".
*   **AST Manipulation (Thao tác trên cây cú pháp):** Để chỉnh sửa code mà không làm hỏng cấu trúc, Onlook sử dụng các trình phân tích AST để thêm/sửa/xóa các thuộc tính Tailwind hoặc các component React một cách an toàn.
*   **Web Container Isolation:** Sử dụng CodeSandbox SDK để chạy dự án trong một môi trường cô lập trên trình duyệt, giúp người dùng xem trước (preview) thay đổi ngay lập tức mà không cần cài đặt môi trường phức tạp.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Visual DOM Editing:** Chỉnh sửa trực tiếp trên iFrame bằng cách bắt các sự kiện kéo thả, thay đổi kích thước và ánh xạ chúng thành các class Tailwind tương ứng.
*   **AI Agent Tools:** AI của Onlook không chỉ viết code mà còn có bộ "Tools" (như Bash read/write, File search, Scrape URL) để hiểu toàn bộ bối cảnh dự án và thực hiện các lệnh phức tạp.
*   **Branching & Checkpoints:** Cho phép người dùng thử nghiệm các thiết kế trên các "nhánh" khác nhau và khôi phục từ các điểm kiểm tra (checkpoints) nếu có lỗi.
*   **Fast Apply:** Kỹ thuật áp dụng các đoạn code do AI tạo ra một cách thông minh, chỉ thay đổi những phần cần thiết thay vì ghi đè toàn bộ file, giúp duy trì cấu trúc code của người dùng.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của Onlook có thể tóm gọn qua 7 bước sau:

1.  **Khởi tạo/Nhập dự án:** Người dùng tạo mới hoặc nhập một dự án Next.js từ GitHub/Local.
2.  **Chạy Sandbox:** Onlook nạp mã nguồn vào một Web Container (thông qua CodeSandbox SDK) để chạy dự án ngay trong trình duyệt.
3.  **Ánh xạ & Chỉ mục (Indexing):** Trình soạn thảo đọc và lập chỉ mục mã nguồn, đồng thời gắn kết các phần tử hiển thị trong iFrame với vị trí code tương ứng.
4.  **Chỉnh sửa trực quan:** Người dùng dùng UI (Figma-like) để kéo thả, sửa style. Các thay đổi này được cập nhật ngay vào iFrame và đồng thời ghi vào file code thông qua trình xử lý AST.
5.  **Tương tác AI:** Người dùng ra lệnh cho AI (ví dụ: "Thêm một section liên hệ"). AI Agent sẽ quét cấu trúc file, đề xuất code và áp dụng vào dự án.
6.  **Kiểm tra & Đồng bộ:** Mọi thay đổi được lưu dưới dạng các checkpoint. Người dùng có thể xem sự khác biệt (diff) giữa code cũ và mới.
7.  **Triển khai (Deployment):** Sau khi hoàn tất, Onlook hỗ trợ deploy dự án lên các nền tảng hosting (như Freestyle) hoặc đẩy PR về GitHub.

**Kết luận:** Onlook không chỉ là một công cụ thiết kế, nó là một **IDE có giao diện thiết kế**, giúp thu hẹp khoảng cách giữa Designer và Developer bằng cách biến Code trở thành ngôn ngữ chung của cả hai.