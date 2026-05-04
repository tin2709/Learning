Dựa trên mã nguồn của dự án **open-slide**, đây là phân tích chi tiết về kiến trúc, công nghệ và luồng hoạt động của framework này:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Runtime:** Xây dựng trên nền tảng **React** (v18+). Điểm đặc biệt là dự án coi Slide là "visual code" – mỗi trang slide thực chất là một React Component tùy ý.
*   **Build Tooling & Dev Server:** Sử dụng **Vite**. Dự án tận dụng tối đa hệ sinh thái plugin của Vite để xử lý Hot Module Replacement (HMR) và module hóa các slide.
*   **Styling:** Sử dụng **Tailwind CSS v4** (bản Alpha/mới nhất) và **Shadcn UI** để xây dựng giao diện người dùng (Inspector, Sidebar).
*   **AST Manipulation (Xử lý mã nguồn):** Sử dụng **Babel (@babel/parser, @babel/types)**. Đây là công nghệ quan trọng nhất giúp framework có thể "đọc" mã React và tự động chèn/sửa các thuộc tính (`style`, `text`, `src`) dựa trên thao tác người dùng trong trình duyệt.
*   **Monorepo Management:** Sử dụng **Turbo** và **pnpm Workspaces** để quản lý đa gói (`core`, `cli`, `demo`, `web`).

### 2. Tư duy Kiến trúc (Architectural Patterns)

*   **Agent-Native Design (Thiết kế hướng Agent):** Khác với các công cụ làm slide thông thường, open-slide được thiết kế để **AI Agent** (như Claude Code) là người viết code chính. Nó cung cấp các "Skills" (dưới dạng Markdown) để Agent hiểu về quy chuẩn canvas và layout.
*   **Fixed Canvas Scaling:** Tư duy thiết kế trên một "khung hình cố định" (**1920x1080**). Framework sử dụng kỹ thuật CSS `transform: scale()` để đảm bảo slide hiển thị đồng nhất trên mọi độ phân giải màn hình mà không cần responsive phức tạp.
*   **Virtual Modules:** Sử dụng tính năng `virtual modules` của Vite (`virtual:open-slide/slides`) để tự động quét thư mục `slides/` và biến các file `.tsx` thành dữ liệu có thể import động vào ứng dụng runtime mà không cần cấu hình thủ công.
*   **Separation of Concerns:**
    *   `packages/cli`: Lo việc khởi tạo project (Scaffolding).
    *   `packages/core`: Chứa logic Runtime (Viewer), Vite Plugins, và CLI thực thi (`dev`, `build`).
    *   `apps/demo`: Môi trường "dogfooding" để kiểm thử framework.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Loc-Tags Injection:** Một Vite plugin tùy chỉnh (`loc-tags-plugin.ts`) sẽ quét mã nguồn slide và tự động chèn thuộc tính `data-slide-loc="line:col"` vào các phần tử JSX. Điều này giúp trình duyệt biết chính xác phần tử đang hiển thị tương ứng với dòng code nào trong file.
*   **Optimistic UI & Buffer Updates:** Trong chế độ Inspector, khi người dùng kéo thanh trượt đổi cỡ chữ hoặc chọn màu, hệ thống sẽ thay đổi trực tiếp DOM (Optimistic). Các thay đổi này được đưa vào một **Buffer**. Chỉ khi người dùng nhấn "Save", Framework mới gọi API backend để ghi đè mã nguồn thật.
*   **BroadcastChannel API:** Sử dụng để đồng bộ hóa trạng thái giữa cửa sổ trình chiếu (Presentation) và cửa sổ người thuyết trình (Presenter View - hiển thị note và slide tiếp theo).
*   **PDF Export via Browser Print:** Tận dụng CSS `@media print` và `window.print()` cùng với việc supersampling (phóng to 2x rồi thu nhỏ 0.5x) để tạo ra PDF chất lượng cao từ các phần tử web phức tạp (như blur, mix-blend-mode).

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Khởi tạo (Init):**
    *   Người dùng chạy `npx @open-slide/cli init`. CLI copy template, thiết lập `package.json` và đặc biệt là các **Agent Skills** vào thư mục `.agents/`.

2.  **Giai đoạn Phát triển (Dev Loop):**
    *   `open-slide dev` khởi động Vite với các plugin tùy chỉnh.
    *   **Plugin Files:** Quét thư mục `slides/`, tạo danh sách slide động.
    *   **Plugin Comments/Edit:** Mở các endpoint API nội bộ (`/__edit`, `/__comments`) để nhận lệnh thay đổi code từ trình duyệt.

3.  **Giai đoạn Tương tác AI (Agent Loop):**
    *   Người dùng ra lệnh cho Agent: "Tạo slide về chủ đề X".
    *   Agent đọc `SKILL.md`, tuân thủ quy tắc canvas 1920x1080, và viết file `.tsx`.
    *   Vite HMR cập nhật giao diện ngay lập tức.

4.  **Giai đoạn Tinh chỉnh (Inspector Loop):**
    *   Người dùng nhấn vào một phần tử trên slide -> Inspector Panel hiện ra.
    *   Người dùng chỉnh sửa hoặc để lại comment (`@slide-comment`).
    *   Mã nguồn được cập nhật trực tiếp bởi framework hoặc bởi Agent thông qua lệnh `/apply-comments`.

5.  **Giai đoạn Xuất bản (Export):**
    *   `open-slide build` đóng gói toàn bộ slide thành một trang HTML tĩnh duy nhất (Single Page App) hoặc file ZIP chứa đầy đủ asset để deploy lên Vercel/Netlify.

### Tổng kết
open-slide không chỉ là một thư viện slide, mà là một **phát triển lại quy trình authoring nội dung thị giác**. Nó xóa bỏ ranh giới giữa "giao diện kéo thả" và "mã nguồn" bằng cách sử dụng AST làm cầu nối, biến mã React thành một cơ sở dữ liệu có thể truy vấn và chỉnh sửa trực quan.