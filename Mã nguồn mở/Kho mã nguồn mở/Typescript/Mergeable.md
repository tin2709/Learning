Dựa trên các tệp tin bạn đã cung cấp, đây là phân tích chi tiết về dự án **Mergeable** - một ứng dụng quản lý Pull Request (PR) thông minh dành cho GitHub.

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án được xây dựng theo mô hình **Monorepo** hiện đại, sử dụng các công nghệ sau:

*   **Quản lý Monorepo:** [PnPM](https://pnpm.io/) (quản lý package) kết hợp với [Turborepo](https://turbo.build/repo) (tối ưu hóa build pipeline).
*   **Frontend (`apps/web`):** 
    *   Framework: **React 18** với **React Router 7** (vừa cập nhật lên framework mode).
    *   Build tool: **Vite**.
    *   UI Kit: **@blueprintjs/core** (phong cách chuyên nghiệp cho dashboard).
    *   Quản lý trạng thái/Data fetching: **TanStack Query (React Query)** với cơ chế Persister để lưu cache vào IndexedDB.
    *   Database cục bộ: **Dexie.js** (IndexedDB wrapper) để lưu cấu hình người dùng và kết nối.
*   **Backend & Telemetry (`apps/backend`, `apps/telemetry`):**
    *   Runtime: **Cloudflare Workers**.
    *   Framework: **Hono** (siêu nhẹ cho edge computing).
    *   ORM/Database: **Prisma** và **Drizzle ORM** kết hợp với **PostgreSQL (Neon serverless)**.
*   **Tài liệu (`apps/docs`):** 
    *   **Astro** với theme **Starlight**.
*   **DevOps & Infrastructure:**
    *   **Docker** & **Helm Chart** (cho việc tự host).
    *   **GitHub Actions:** Tự động hóa kiểm thử, build Docker image, và deploy lên Cloudflare.
    *   **Cosign:** Ký số cho các Docker image để đảm bảo bảo mật.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architecture)

*   **Kiến trúc Local-first (Ưu tiên lưu trữ cục bộ):** Đây là điểm nổi bật nhất. Mọi dữ liệu nhạy cảm (như GitHub Token) và dữ liệu PR được lưu trực tiếp trong trình duyệt người dùng qua IndexedDB (`lib/db.ts`). Mergeable không giữ Token của bạn trên server của họ trừ khi bạn sử dụng luồng OAuth.
*   **Tách biệt Logic GitHub:** Client GitHub (`lib/github/client.ts`) được xây dựng tách biệt, hỗ trợ cả REST API và GraphQL. Điều này cho phép ứng dụng lấy dữ liệu chi tiết (như trạng thái Review, Check CI, Discussions) một cách hiệu quả nhất.
*   **Cơ chế "Attention Set" (Bộ tập trung):** Tư duy kiến trúc ở đây là giải quyết bài toán "Đến lượt ai?". Logic này được đóng gói trong `lib/github/attention.ts`, tự động tính toán trạng thái PR dựa trên các sự kiện (comments mới, thay đổi trạng thái CI, hoặc yêu cầu review).
*   **Mở rộng qua Micro-packages:** Các cấu hình dùng chung như ESLint, TypeScript, và các Plugin Vite tùy chỉnh được tách ra thành các package riêng trong thư mục `packages/` để tái sử dụng giữa web, backend và telemetry.

---

### 3. Các kỹ thuật chính nổi bật (Key Noteworthy Techniques)

*   **Phân tích truy vấn Search phức tạp:** Dự án tự xây dựng một parser cho GitHub Search Query (`lib/github/search.ts`). Nó cho phép người dùng nhập các chuỗi tìm kiếm phức tạp, hỗ trợ cả toán tử loại trừ (`-`), khoảng giá trị (`..`), và ghép nhiều query bằng dấu chấm phẩy (`;`).
*   **Xử lý đa kết nối (Multi-instance):** Hỗ trợ kết nối đồng thời nhiều tài khoản GitHub hoặc các instance GitHub Enterprise khác nhau. Dữ liệu từ tất cả các nguồn sẽ được tổng hợp lại trên một Dashboard duy nhất.
*   **Sử dụng Web Workers:** Telemetry (thu thập dữ liệu sử dụng ẩn danh) được thực hiện trong Web Worker (`apps/web/src/worker.ts`) để không làm ảnh hưởng đến hiệu suất của luồng UI chính.
*   **Testing với PollyJS:** Trong thư mục test, dự án sử dụng `PollyJS` để "record & replay" các HTTP requests. Kỹ thuật này giúp các unit test chạy cực nhanh và ổn định vì không cần thực sự gọi đến API của GitHub mỗi lần chạy test.
*   **Dynamic Environment Variables:** Sử dụng một plugin Vite tự viết (`packages/vite-plugin-process-env`) để inject biến môi trường vào runtime, đặc biệt hữu ích khi chạy trong Docker/Nginx nơi các biến môi trường cần được thay đổi mà không cần build lại mã nguồn.

---

### 4. Tóm tắt luồng hoạt động của dự án

1.  **Khởi tạo:** Người dùng truy cập ứng dụng, thêm kết nối (Connection) bằng Personal Access Token hoặc qua OAuth (Backend sẽ xử lý login và redirect lại token).
2.  **Lấy dữ liệu:** 
    *   Ứng dụng sử dụng các truy vấn GraphQL định nghĩa trong `search.graphql` để lấy danh sách PR từ GitHub theo các Section (Tab) mà người dùng cấu hình.
    *   Dữ liệu được lưu tạm vào IndexedDB thông qua cơ chế cache của React Query.
3.  **Xử lý:**
    *   Hệ thống tính toán kích thước PR (XS, S, M, L, XL) dựa trên số dòng thay đổi (`lib/size.ts`).
    *   Thuật toán Attention Set sẽ quét các thảo luận (discussions) và trạng thái review để đánh dấu các PR cần người dùng xử lý ngay lập tức.
4.  **Hiển thị:** Dashboard hiển thị các Section với khả năng lọc, tìm kiếm mờ (fuzzy search), và phím tắt (Hotkeys) để điều hướng nhanh.
5.  **Hậu trường:** Một worker chạy ngầm thực hiện gửi telemetry (nếu không bị tắt) và làm mới dữ liệu định kỳ mỗi 5 phút.

Dự án này là một ví dụ mẫu mực về việc xây dựng một **Developer Tool** hiện đại: coi trọng quyền riêng tư (local-first), kiến trúc monorepo sạch sẽ và trải nghiệm người dùng tối ưu.