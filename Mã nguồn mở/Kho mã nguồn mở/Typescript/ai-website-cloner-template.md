Dựa trên mã nguồn của kho lưu trữ **AI Website Cloner Template**, dưới đây là phân tích chi tiết về dự án này dưới bốn góc độ quan trọng:

### 1. Công nghệ cốt lõi (Core Technology)

Dự án này sử dụng những công nghệ tiên tiến nhất (thậm chí là bleeding-edge) trong hệ sinh thái Web và AI:

*   **Next.js 16 & React 19:** Sử dụng phiên bản mới nhất hỗ trợ App Router và React Server Components (RSC). Đây là nền tảng mạnh mẽ nhất để xây dựng web hiện đại.
*   **Tailwind CSS v4:** Tận dụng công cụ styling thế hệ mới nhất với hiệu suất cao và khả năng cấu hình bằng biến CSS trực tiếp, hỗ trợ dải màu OKLCH.
*   **shadcn/ui (Base Nova):** Sử dụng các primitive từ `@base-ui/react` (thay vì Radix cũ) để tạo ra các component UI có độ tùy biến cực cao nhưng vẫn nhẹ.
*   **AI Agent Orchestration:** Dự án không chỉ là mã nguồn web, mà là một "hệ điều hành" cho các AI Agent (như Claude Code, Cursor, Cline). Nó hỗ trợ **MCP (Model Context Protocol)** để AI có thể điều khiển trình duyệt (Chrome) nhằm trích xuất dữ liệu thực tế.
*   **Docker:** Hỗ trợ môi trường phát triển (Dockerfile.dev) và sản xuất (standalone mode) đồng nhất.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của dự án được thiết kế theo mô hình **"Foreman & Specialist" (Quản đốc và Chuyên gia)**:

*   **Single Source of Truth (SSOT):** Toàn bộ chỉ dẫn cho các loại AI khác nhau được quản lý tại một nơi duy nhất (`AGENTS.md` và `SKILL.md`). Các script tự động (`sync-agent-rules.sh`, `sync-skills.mjs`) sẽ đồng bộ hóa các chỉ dẫn này sang định dạng riêng của từng nền tảng (JSON, TOML, Markdown).
*   **Pipeline Đa giai đoạn:** Quá trình clone được chia làm 5 bước độc lập: Reconnaissance (Trinh sát) -> Foundation (Xây móng) -> Spec Writing (Viết đặc tả) -> Parallel Build (Xây dựng song song) -> Assembly & QA (Lắp ráp và Kiểm định).
*   **Kiến trúc dựa trên Đặc tả (Spec-driven Architecture):** AI không được phép code ngay. Nó phải viết file `.spec.md` cho từng component dựa trên các giá trị CSS thực tế (`getComputedStyle()`). File này đóng vai trò là "bản thiết kế" để các AI builder khác thực thi chính xác.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Parallelization via Git Worktrees:** Kỹ thuật cực kỳ sáng tạo: AI sẽ tạo các **Git Worktree** riêng biệt cho từng section của website. Điều này cho phép nhiều "AI Worker" xây dựng các phần khác nhau cùng lúc mà không gây xung đột code (merge conflicts) trên nhánh chính.
*   **Computed Style Extraction:** Sử dụng các đoạn script JavaScript tiêm vào trình duyệt (qua Chrome MCP) để đọc chính xác thông số pixel, font chữ, màu sắc thực tế của trang web mục tiêu thay vì để AI tự đoán (hallucination).
*   **Cross-Platform Instruction Syncing:** Kỹ thuật metaprogramming nhẹ (dùng Bash/Node) để tự động tạo ra `.clinerules`, `.cursor/rules`, `.amazonq/rules` từ file gốc, giúp dự án tương thích với hàng chục công cụ AI khác nhau mà không cần bảo trì thủ công từng file.
*   **Type-Safe UI Building:** Ép buộc sử dụng TypeScript Strict Mode và định nghĩa Interface cho mọi cấu trúc nội dung trích xuất được.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý từ khi người dùng nhập lệnh `/clone-website` diễn ra như sau:

1.  **Giai đoạn Trinh sát (Recon):** AI mở URL mục tiêu, chụp ảnh toàn màn hình ở nhiều breakpoint (Desktop, Tablet, Mobile) và trích xuất "Design Tokens" (màu sắc, font chữ).
2.  **Giai đoạn Nền móng (Foundation):** Hệ thống tự động cập nhật `globals.css` và `layout.tsx`, tải toàn bộ tài sản (ảnh, video, SVG) về thư mục `public/`.
3.  **Giai đoạn Phân rã (Decomposition):** AI phân tích cấu trúc DOM, chia trang web thành các section và viết file spec chi tiết cho mỗi section vào `docs/research/components/`.
4.  **Giai đoạn Xây dựng song song (Build):** Một AI "Quản đốc" điều phối các "AI Builder" làm việc trên các worktree riêng để code các file `.tsx`. Mỗi builder chỉ tập trung vào 1 file spec.
5.  **Giai đoạn Tổng hợp & QA:** AI gộp tất cả các worktree lại, lắp ráp vào `page.tsx` và thực hiện so sánh Visual Diff (ảnh chụp màn hình bản clone vs bản gốc) để tìm sai sót.

### Tổng kết
Đây là một dự án **"Infrastructure-as-Instructions"** (Hạ tầng dựa trên chỉ dẫn). Sức mạnh của nó không nằm ở 10-20 file code React ban đầu, mà nằm ở **hệ thống quy tắc và pipeline** ép buộc AI phải làm việc có kỷ luật, trích xuất dữ liệu khoa học để tạo ra sản phẩm "Pixel Perfect".