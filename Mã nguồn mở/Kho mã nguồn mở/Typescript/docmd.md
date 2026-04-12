Dưới đây là bản phân tích chuyên sâu về hệ thống **docmd** dựa trên mã nguồn bạn đã cung cấp. Đây là một dự án hiện đại, tinh gọn và thể hiện tư duy kiến trúc hướng tới hiệu năng cực cao.

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

`docmd` không sử dụng các Framework nặng nề như React hay Vue để sinh site (SSG), thay vào đó nó chọn hướng **"Vanilla-first"**:

*   **Runtime:** Node.js (ES Modules).
*   **Markdown Engine:** `markdown-it` làm hạt nhân, được tùy biến mạnh mẽ qua hệ thống plugin.
*   **Templating Engine:** `lite-template` (một biến thể siêu nhẹ của EJS).
*   **Frontmatter:** `lite-matter` để phân tích metadata trong file Markdown.
*   **Build Tool:** `esbuild` được dùng để đóng gói và minify tài nguyên (JS/CSS) với tốc độ cực nhanh.
*   **Communication:** WebSockets (`ws`) được sử dụng trong chế độ Development để hỗ trợ Hot Reload và giao tiếp RPC giữa Browser và Server.
*   **Bundling:** Kiến trúc Monorepo quản lý bằng `pnpm workspaces`.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `docmd` dựa trên 4 trụ cột chính:

#### A. Isomorphic Logic (Logic đồng nhất)
Mã nguồn được thiết kế để chạy được ở cả **Node.js (CLI)** và **Browser (Live Editor)**. Gói `@docmd/parser` hoàn toàn tách biệt khỏi các API của hệ điều hành (như `fs`), cho phép nó render Markdown thành HTML ngay trên trình duyệt mà không cần server.

#### B. Zero-Config & Auto-Routing
Tư duy "Convention over Configuration". Nếu người dùng không cấu hình, `auto-router.ts` sẽ tự động quét cấu trúc thư mục, trích xuất tiêu đề từ H1 hoặc Frontmatter để xây dựng cây điều hướng (Navigation Tree) một cách thông minh.

#### C. Plugin-Driven Architecture
Hệ thống plugin của `docmd` rất linh hoạt, chia làm 2 loại:
1.  **Build-time Hooks:** Can thiệp vào quá trình render (thêm thẻ meta, script, sitemap).
2.  **Runtime Actions (RPC):** Cho phép trình duyệt "gọi" các hàm Node.js thông thông qua WebSocket (ví dụ: plugin `threads` dùng để ghi comment trực tiếp vào file Markdown).

#### D. Static-First (No Hydration Gap)
Khác với Docusaurus (React), `docmd` sinh ra HTML thuần. Điều này loại bỏ hoàn toàn quá trình "Hydration" ở client, giúp trang web có điểm Performance gần như tuyệt đối (100/100) và SEO cực tốt.

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

#### A. Kỹ thuật "Depth-Tracking" trong Parser
Trong `common-containers.ts`, tác giả triển khai một thuật toán theo dõi độ sâu (depth tracking) để cho phép lồng các container (`:::`) vào nhau một cách vô hạn mà không làm gãy cấu trúc Markdown. Điều này cực kỳ khó thực hiện với các regex đơn giản.

#### B. Source Mapping & Reverse Editing
Gói `source-tools.ts` chứa các kỹ thuật xử lý chuỗi cấp cao. Nó có khả năng:
*   Xác định vị trí dòng/cột chính xác của một đoạn văn bản trong file gốc dựa trên bản render.
*   Thực hiện các thao tác `wrapText`, `replaceBlock`, `insertAfter` trực tiếp vào file nguồn. Đây là nền tảng cho tính năng "Live Editing".

#### C. Quy trình chuẩn hóa Config (Schema Normalization)
Hệ thống sử dụng `config-schema.ts` để map các cấu hình cũ (Legacy V1/V2) sang cấu hình mới (V3). Kỹ thuật này giúp duy trì tính tương thích ngược (Backward Compatibility) mà không làm bẩn mã nguồn chính.

#### D. Virtual Template Bundling (trong `live/build.ts`)
Khi build bản Live Editor, tác giả sử dụng một plugin `esbuild` tùy chỉnh để biến các file template EJS thành các **Virtual Modules**. Điều này cho phép trình duyệt "require" các template như thể chúng là file JS.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Build (Build Pipeline)
1.  **Load Config:** Đọc `docmd.config.js`, nếu không có thì chạy `auto-router`.
2.  **Plugin Setup:** Kích hoạt các plugin (Search, SEO, Mermaid...).
3.  **Asset Preparation:** Copy CSS/JS từ `@docmd/ui` và `@docmd/themes` vào thư mục output.
4.  **The Versioning Loop:** Nếu có nhiều phiên bản, hệ thống lặp qua từng thư mục phiên bản.
5.  **Markdown Processing:** 
    *   Parse Frontmatter.
    *   Render Markdown -> HTML.
    *   Trích xuất Headings để làm TOC (Table of Contents).
6.  **Template Injection:** Đổ dữ liệu HTML vào layout EJS.
7.  **Post-Build:** Sinh `sitemap.xml`, `search-index.json`, và `llms.txt`.

#### B. Luồng Development (Dev Server)
1.  Khởi tạo HTTP Server phục vụ file tĩnh.
2.  Thiết lập `fs.watch` trên toàn bộ thư mục `docs/` và `assets/`.
3.  Khi có thay đổi: Re-build cực bộ -> Gửi tín hiệu `reload` qua WebSocket.
4.  Đặc biệt: Nếu trình duyệt gửi một `Action Call` (như lưu comment), `action-dispatcher.ts` sẽ nhận, thực thi ghi file trên đĩa cứng, và ra lệnh cho trình duyệt cập nhật lại UI.

---

### Tổng kết

`docmd` là một minh chứng cho việc **"Less is More"**. 
*   **Về mặt kỹ thuật:** Nó giải quyết bài toán hiệu năng bằng cách loại bỏ các lớp trừu tượng dư thừa (Framework JS).
*   **Về mặt trải nghiệm:** Nó tối ưu cho cả người đọc (tốc độ) và người viết (Live Editor, AI context ready).
*   **Điểm sáng nhất:** Khả năng xử lý file Markdown như một Database (thông qua `source-tools`) biến nó từ một SSG thông thường thành một CMS dạng file-based mạnh mẽ.