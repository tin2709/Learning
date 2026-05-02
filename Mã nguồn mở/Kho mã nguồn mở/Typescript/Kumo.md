Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **Cloudflare Kumo**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và quy trình vận hành của hệ thống thiết kế (Design System) này.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Kumo được xây dựng trên một ngăn xếp công nghệ hiện đại, tối ưu cho hiệu suất và khả năng mở rộng:

*   **Framework Foundation:** **React** (v18/19) kết hợp với **Base UI** (một thư viện unstyled components từ MUI). Việc sử dụng Base UI giúp dự án tập trung vào giao diện (styling) mà không cần lo lắng về logic tiếp cận (accessibility) phức tạp như quản lý focus hay phím tắt.
*   **Styling System:** **Tailwind CSS v4**. Đây là một trong những dự án tiên phong sử dụng Tailwind v4, tận dụng tính năng `@theme` mới và khả năng xử lý CSS biến (Custom Properties) mạnh mẽ.
*   **Build & Bundling:** **Vite** được sử dụng ở chế độ Library Mode để tạo ra các gói ESM có khả năng tree-shaking cao. **Astro** được dùng cho trang tài liệu (docs) để tối ưu tốc độ load trang tĩnh.
*   **Linter & Quality:** Sử dụng **oxlint** (một linter viết bằng Rust siêu nhanh) kết hợp với các quy tắc tùy chỉnh (custom rules) để ép buộc các tiêu chuẩn của hệ thống thiết kế.
*   **Infrastructure:** Chạy trên **Cloudflare Workers** (cho docs site và screenshot worker) và sử dụng **pnpm workspaces** để quản lý monorepo.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Kumo được thiết kế theo mô hình **"Source of Truth" (Nguồn sự thật duy nhất)** và **"Headless-First"**:

*   **Cấu trúc 3 lớp (Primitives -> Components -> Blocks):**
    *   *Primitives:* Tái xuất bản trực tiếp Base UI (unstyled).
    *   *Components:* Các thành phần UI cơ bản được Cloudflare style hóa (Button, Input).
    *   *Blocks:* Các khuôn mẫu cấp cao hơn (PageHeader, ResourceList) được cài đặt qua CLI thay vì import trực tiếp để tránh phình bundle.
*   **Registry-Driven:** Trái tim của hệ thống là `component-registry.json`. File này chứa toàn bộ metadata của component (props, variants, code examples). Nó được tự động sinh ra và cung cấp dữ liệu cho Docs site, CLI, và các công cụ AI (như Claude).
*   **Token-Based Theming:** Thay vì dùng màu cứng (hardcoded), Kumo sử dụng hệ thống **Semantic Tokens** (`bg-kumo-base`, `text-kumo-default`). Khả năng hỗ trợ Dark Mode là hoàn toàn tự động thông qua hàm `light-dark()` của CSS hiện đại, giúp loại bỏ việc dùng prefix `dark:` trong class name.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Automated Codegen (Sinh mã tự động):** 
    *   Tự động sinh các tệp re-export cho primitives.
    *   Tự động trích xuất các variant từ code TypeScript để đồng bộ hóa với Figma.
    *   Tự động sinh CSS theme từ cấu hình TypeScript (`theme-generator`).
*   **Custom Linting Rules (Oxlint Plugins):** Dự án viết các plugin cho linter để ngăn chặn các "anti-pattern":
    *   `no-primitive-colors`: Cấm sử dụng màu Tailwind mặc định (như `bg-blue-500`), bắt buộc dùng token Kumo.
    *   `no-tailwind-dark-variant`: Cấm dùng `dark:`, bắt buộc dùng hệ thống theme tự động.
    *   `enforce-variant-standard`: Ép buộc cách đặt tên các biến variant (`KUMO_{NAME}_VARIANTS`).
*   **Scaffolding (Plop.js):** Sử dụng `plop` để tạo component mới. Việc này đảm bảo mọi component đều có đầy đủ index, stories, tests và được đăng ký tự động vào build config và package.json.
*   **Visual Regression Testing:** Sử dụng một Cloudflare Worker riêng (`kumo-screenshot-worker`) để chụp ảnh component và so sánh pixel (pixelmatch) giữa các pull request, đảm bảo không có thay đổi giao diện ngoài ý muốn.

---

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Quy trình phát triển Component:
1.  Developer chạy `pnpm new:component`.
2.  Viết code tại `src/components/{name}.tsx`.
3.  Định nghĩa các variant và styles bằng Tailwind v4.
4.  Linter kiểm tra xem có dùng màu primitive hay không.
5.  Build script chạy, trích xuất metadata vào Registry.

#### B. Luồng Codegen & Docs:
1.  Script `extract-demo-examples.ts` quét các file demo trong Docs site.
2.  `schema-generator.ts` phân tích kiểu dữ liệu TypeScript của Props.
3.  Tất cả được gộp vào `component-registry.json`.
4.  Astro Docs site đọc Registry này để hiển thị bảng Props và Code Snippets một cách tự động.

#### C. Quy trình CI/CD:
1.  **Commit:** Lefthook kiểm tra Changesets (bắt buộc phải có mô tả thay đổi).
2.  **Pull Request:** 
    *   Build package và deploy bản preview cho Docs site.
    *   Chạy screenshot worker để kiểm tra lỗi hiển thị (Visual Regression).
    *   Tạo bản phát hành beta (`-beta.{hash}`) lên NPM thông qua `pkg.pr.new`.
3.  **Release:** Changeset bot tự động gộp các thay đổi, nâng version và publish bản chính thức.

### Tổng kết
Kumo là một ví dụ điển hình về việc xây dựng hệ thống thiết kế **"Machine-readable"**. Nó không chỉ phục vụ con người mà còn được cấu trúc hóa cực tốt để các công cụ tự động (Linter, Codegen, AI) có thể tham gia sâu vào quy trình phát triển, giúp duy trì tính nhất quán tuyệt đối trên quy mô lớn của Cloudflare.