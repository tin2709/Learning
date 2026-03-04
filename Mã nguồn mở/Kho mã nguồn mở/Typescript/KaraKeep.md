Dựa trên cấu trúc thư mục và mã nguồn của dự án **Karakeep** (trước đây là Hoarder), dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và luồng hoạt động của hệ thống này.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Karakeep được xây dựng theo mô hình **Monorepo** sử dụng **Turborepo** và **pnpm**, cho phép chia sẻ mã nguồn giữa Web, Mobile, CLI và Extension một cách hiệu quả.

*   **Ngôn ngữ chính:** TypeScript (bao phủ gần 99% dự án).
*   **Frontend (Web):** Next.js (App Router), Tailwind CSS, Radix UI (thông qua shadcn/ui).
*   **Mobile:** React Native sử dụng **Expo** và NativeWind (Tailwind cho Mobile).
*   **Backend & API:**
    *   **tRPC:** Đây là "xương sống" giao tiếp, cung cấp Type-safety từ server đến mọi client (Web, Mobile, CLI, Extension).
    *   **Hono:** Framework siêu nhẹ dùng cho phần API route.
*   **Cơ sở dữ liệu & Search:**
    *   **Drizzle ORM:** Dùng để tương tác với database (thường là PostgreSQL hoặc SQLite).
    *   **Meilisearch:** Đảm nhiệm tính năng tìm kiếm toàn văn (Full-text search) tốc độ cao.
*   **Xử lý nền (Background Processing):**
    *   **Puppeteer:** Để crawl dữ liệu web và chụp ảnh màn hình.
    *   **yt-dlp:** Lưu trữ video từ các nền tảng như YouTube.
    *   **Monolith:** Lưu trữ toàn bộ trang HTML để tránh "link rot" (mất nội dung gốc).
*   **AI & Inference:** Tích hợp OpenAI API hoặc **Ollama** (cho các mô hình chạy local) để tự động gắn thẻ (tagging) và tóm tắt nội dung.

---

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Dự án tuân thủ triết lý **"Self-hosting first"** và **"Type-safe everything"**.

#### a. Cấu trúc Monorepo (`apps/` và `packages/`)
*   **`packages/trpc`**: Chứa toàn bộ logic nghiệp vụ (Business Logic). Mọi thao tác như tạo bookmark, quản lý list đều nằm ở đây. Điều này giúp CLI (`apps/cli`) và Extension (`apps/browser-extension`) có thể gọi chung một hàm như Web app.
*   **`packages/db`**: Định nghĩa Schema duy nhất bằng Drizzle, đảm bảo tính nhất quán của dữ liệu trên toàn bộ hệ thống.
*   **`apps/workers`**: Tách biệt phần xử lý nặng (crawling, AI, OCR) khỏi API chính để đảm bảo Web UI luôn mượt mà.

#### b. Thiết kế "Local-first" & Plugin
Hệ thống cho phép người dùng cấu hình linh hoạt: dùng OpenAI nếu muốn chất lượng cao, hoặc dùng Ollama nếu muốn bảo mật dữ liệu hoàn toàn trên máy chủ cá nhân.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Type-safe API với tRPC:** Karakeep không định nghĩa các endpoint REST truyền thống một cách thủ công. Thay vào đó, tRPC tự động xuất (export) các kiểu dữ liệu từ server sang client. Khi bạn đổi tên một trường trong database, TypeScript sẽ báo lỗi ngay lập tức ở code giao diện Mobile hay CLI.
*   **Crawl & Metadata Extraction:** Sử dụng một quy trình đa tầng:
    1.  Lấy metadata cơ bản (OpenGraph tags).
    2.  Dùng Puppeteer để render JavaScript (với các trang SPA).
    3.  Dùng `monolith` để đóng gói toàn bộ tài nguyên trang web vào một file duy nhất.
*   **Hệ thống Rule Engine:** (Nằm trong `packages/trpc/lib/ruleEngine.ts`) Cho phép người dùng thiết lập logic tự động: "Nếu link từ domain X, tự động cho vào List Y và gắn Tag Z".
*   **Quản lý phiên bản Database:** Sử dụng Drizzle Kit để tạo các file migration SQL (trong `packages/db/drizzle/`), giúp việc nâng cấp server của người dùng tự động và an toàn.

---

### 4. Luồng hoạt động (Operational Flow)

Lấy ví dụ luồng **"Lưu một liên kết"**:

1.  **Trigger (Client):** Người dùng nhấn "Save" trên Browser Extension hoặc chia sẻ link vào app Mobile. Client gửi một request tRPC `bookmarks.createBookmark`.
2.  **API Handling (Server):** 
    *   Server kiểm tra quyền hạn (Auth).
    *   Lưu thông tin sơ bộ (URL) vào Database.
    *   Đẩy một job vào hàng đợi (Queue).
3.  **Worker Processing (Background):**
    *   **Worker Crawl:** Puppeteer khởi động, truy cập URL, lấy tiêu đề, mô tả và chụp ảnh màn hình/lưu HTML.
    *   **Worker AI:** Nếu bật AI, nội dung văn bản sau khi crawl sẽ được gửi đến OpenAI/Ollama để phân tích tag và tóm tắt.
    *   **Worker Search:** Cập nhật chỉ mục vào Meilisearch.
4.  **Sync:** Web UI nhận được tín hiệu (thông qua tRPC Invalidation hoặc polling) và hiển thị Bookmark đã được làm giàu thông tin (có ảnh, có tag, có tóm tắt).

---

### 5. Đánh giá về mặt kỹ thuật

*   **Ưu điểm:** Kiến trúc cực kỳ hiện đại, sử dụng những công nghệ tốt nhất hiện nay trong hệ sinh thái TypeScript. Khả năng tái sử dụng code rất cao nhờ Monorepo. Việc tách biệt Worker giúp hệ thống chịu tải tốt và dễ mở rộng.
*   **Thách thức:** Do sử dụng Puppeteer và các tiến trình AI, Karakeep yêu cầu cấu hình RAM khá lớn khi tự host (thường tối thiểu 2GB+). Việc quản lý Docker với nhiều dịch vụ đi kèm (Redis, Meilisearch, Chrome) đòi hỏi người dùng có kiến thức cơ bản về DevOps.

Karakeep là một ví dụ điển hình về cách xây dựng một ứng dụng Full-stack hiện đại, bài bản và có tính thực tiễn cao.