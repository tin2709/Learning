Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Intlayer**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Intlayer là một hệ sinh thái i18n (đa ngôn ngữ) hiện đại, tập trung vào trải nghiệm lập trình viên (DX) và hiệu suất:

*   **Ngôn ngữ chính:** **TypeScript (98.8%)** - Tận dụng tối đa hệ thống Type để cung cấp tính năng autocompletion cho các khóa dịch (dictionaries).
*   **Runtime & Package Manager:** **Bun** - Dự án đã chuyển từ pnpm sang Bun để tối ưu tốc độ cài đặt và thực thi script.
*   **Xây dựng hệ thống (Build System):** **Turborepo** được dùng để quản lý monorepo, giúp song song hóa việc build/test giữa các package (`@intlayer/core`, `@intlayer/cli`, v.v.).
*   **Compiler & Transformation:** Sử dụng **SWC (Rust)** và **Babel** để viết các plugin chuyển đổi mã nguồn, tự động trích xuất nội dung từ các component mà không làm tăng kích thước bundle (zero-runtime overhead).
*   **AI Integration:** Tích hợp các mô hình LLM (OpenAI, Anthropic, Gemini) thông qua AI SDK để tự động dịch thuật và kiểm tra tính nhất quán của tài liệu.
*   **Giao thức MCP (Model Context Protocol):** Cung cấp server MCP cho phép các AI Agent (như Claude Code) tự động hiểu và chỉnh sửa nội dung đa ngôn ngữ trong project.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Intlayer phá vỡ cách tiếp cận i18n truyền thống:

*   **Per-component i18n (Local Dictionaries):** Thay vì sử dụng một file JSON khổng lồ (`en.json`), Intlayer khuyến khích khai báo nội dung ngay bên cạnh component (`*.content.ts`). Điều này giúp code sạch hơn, dễ bảo trì và hỗ trợ **Tree-shaking** cực tốt (chỉ load những gì cần thiết cho trang đó).
*   **Framework Agnostic Core:** Lõi logic nằm ở package `@intlayer/core`, trong khi các adapter riêng biệt (`next-intlayer`, `react-intlayer`, `vue-intlayer`, `svelte-intlayer`) đảm bảo tính tương thích với mọi môi trường.
*   **Hybrid Content Management:** Hỗ trợ song song cả **Local Files** (Git-based) và **Remote CMS**. Lập trình viên có thể khai báo nội dung bằng mã (TypeScript), nhưng người viết nội dung (Copywriter) có thể sửa chúng qua Visual Editor.
*   **Type-Safe Routing:** Kiến trúc định tuyến (routing) tích hợp sẵn locale prefix (ví dụ: `/fr/dashboard`) và tự động xử lý SEO metadata cho từng ngôn ngữ.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Module Augmentation:** Sử dụng kỹ thuật mở rộng module của TypeScript để khi lập trình viên định nghĩa một dictionary mới, hệ thống sẽ tự động cập nhật Type toàn cục, cho phép IDE gợi ý code (IntelliSense) chính xác từng khóa dịch.
*   **Plugin System:** Hệ thống core sử dụng pattern "Deep Transform Plugins" để xử lý các logic phức tạp như lọc locale, ẩn/hiện nội dung theo điều kiện (Condition), hoặc xử lý số nhiều (Enumeration).
*   **Isomorphic Fetching:** Các hàm trích xuất dữ liệu được thiết kế để chạy được cả trên Server (Node/Bun) và Client (Browser), tối ưu cho Next.js Server Components.
*   **Chokidar Watcher:** Trong `@intlayer/chokidar`, hệ thống sử dụng cơ chế lắng nghe thay đổi file thời gian thực để rebuild các dictionary ngay khi lập trình viên vừa lưu code.
*   **Proxy Pattern:** Sử dụng Proxy trong Next.js (`proxy.ts`) để điều hướng yêu cầu người dùng đến đúng locale mà không làm hỏng logic định tuyến mặc định của framework.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khai báo (Declaration):** Người dùng tạo file `.content.ts` (ví dụ: `button.content.ts`).
2.  **Trích xuất & Biên dịch (Extraction & Compilation):** Khi chạy lệnh `intlayer build`, CLI/Plugins sẽ quét toàn bộ project, tìm các file content, validate cấu trúc qua Zod/TypeScript, và tạo ra các tệp dictionary tĩnh (JSON/TS) trong thư mục `.intlayer`.
3.  **Tạo Type (Type Generation):** Hệ thống tự động tạo ra các file định nghĩa `.d.ts` dựa trên nội dung vừa biên dịch để đảm bảo Type-safety.
4.  **Sử dụng (Consumption):**
    *   **Client side:** Sử dụng hook `useIntlayer('key')`.
    *   **Server side:** Sử dụng hàm `getIntlayer('key')`.
5.  **AI Lifecycle (Tùy chọn):** Lệnh `intlayer fill` gửi các đoạn nội dung chưa có bản dịch lên AI để tự động điền (Auto-fill), sau đó Format lại code bằng Biome/Prettier.
6.  **Visual Editing:** Nếu bật `liveSync`, Visual Editor sẽ mở ra một iframe bao bọc ứng dụng, cho phép người dùng click trực tiếp vào văn bản trên web để sửa, và thay đổi sẽ được ghi ngược lại vào file mã nguồn hoặc database của CMS.

**Tóm lại:** Intlayer không chỉ là một thư viện dịch thuật, mà là một **"Hạ tầng nội dung" (Content Infrastructure)** giúp hợp nhất quy trình làm việc giữa Lập trình viên, Máy dịch (AI) và Người quản lý nội dung.