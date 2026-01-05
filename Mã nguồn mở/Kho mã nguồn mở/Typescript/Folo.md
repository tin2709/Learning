Dưới đây là bản phân tích chi tiết về dự án **Folo (RSSNext/Follow)** dựa trên cấu trúc mã nguồn và các tệp cấu hình mà bạn đã cung cấp.

---

# Báo cáo Phân tích Kỹ thuật Dự án Folo

## 1. Công nghệ cốt lõi (Core Tech Stack)
Dự án được xây dựng theo mô hình **Monorepo** hiện đại, tối ưu hóa việc chia sẻ mã nguồn giữa các nền tảng Desktop, Mobile và Web.

*   **Quản lý Monorepo:** `pnpm workspaces` kết hợp với `Turbo` (Turborepo) để quản lý build pipeline và caching.
*   **Frontend Framework:** 
    *   **React 19:** Sử dụng phiên bản mới nhất cho cả Web và Desktop.
    *   **Expo / React Native:** Cho ứng dụng di động (iOS/Android).
*   **Desktop App:** **Electron** kết hợp với **Vite** (`electron-vite`) để tối ưu hiệu suất renderer.
*   **Quản lý trạng thái (State Management):**
    *   **Jotai:** Quản lý state dạng nguyên tử (atomic) cho các tương tác nhỏ.
    *   **Zustand:** Quản lý các store phức tạp hơn.
    *   **TanStack Query (React Query):** Quản lý trạng thái từ server và caching dữ liệu bài viết.
*   **Cơ sở dữ liệu (Local-first):** **Drizzle ORM** kết hợp với **SQLite** (SQLite cho desktop và Expo SQLite cho mobile). Đây là tư duy thiết kế bài bản để hỗ trợ đọc offline.
*   **Styling:** 
    *   **Tailwind CSS:** Cho Web/Desktop.
    *   **Nativewind (Tailwind for React Native):** Cho Mobile.
    *   **Framer Motion (LazyMotion):** Xử lý chuyển động mượt mà trên desktop.

---

## 2. Kỹ thuật và Tư duy Kiến trúc (Architecture Thinking)

### A. Kiến trúc Local-first và Đồng bộ
Kiến trúc của Folo không chỉ đơn thuần là gọi API rồi hiển thị. Nó ưu tiên lưu trữ tại local (`packages/internal/database`). Việc sử dụng Drizzle giúp định nghĩa Schema chung cho cả Desktop và Mobile, đảm bảo tính nhất quán của dữ liệu unread, starred, và lịch sử đọc.

### B. Hybrid Renderer (Desktop)
Phần Desktop (`apps/desktop`) được thiết kế cực kỳ linh hoạt:
*   Vừa có thể chạy như một ứng dụng Electron hoàn chỉnh (truy cập API hệ thống, Tray, Shortcuts).
*   Vừa có thể chạy như một Web App (SPA) thông thường trên trình duyệt.
*   Kiến trúc tách biệt `layer/main` (Node.js) và `layer/renderer` (Web Standard) giúp dễ dàng bảo trì.

### C. Tận dụng AI làm trọng tâm (AI-Centric)
Dự án tích hợp AI sâu vào luồng trải nghiệm người dùng:
*   **AI Summary & Translation:** Không chỉ là tính năng thêm vào, AI được sử dụng để tóm tắt nội dung ngay trong timeline, dịch thuật đa ngôn ngữ và thậm chí sắp xếp timeline theo sở thích người dùng (AI Timeline Sort).
*   **BYOK (Bring Your Own Key):** Cho phép người dùng sử dụng API Key cá nhân, giảm chi phí vận hành cho dự án mã nguồn mở.

---

## 3. Các kỹ thuật chính nổi bật (Key Implementation Techniques)

*   **Hot Update cho Renderer:** Desktop có một cơ chế `hot-updater.ts` riêng biệt. Nó cho phép cập nhật phần giao diện (Vite bundle) mà không cần người dùng phải tải lại toàn bộ file cài đặt Electron (thường rất nặng).
*   **Xử lý nội dung bài viết (Readability):** Sử dụng thư viện `@follow-app/readability` để trích xuất nội dung chính từ các link web, loại bỏ quảng cáo và các thành phần thừa, tạo trải nghiệm đọc "sạch".
*   **Hệ thống Icon thông minh:** Ưu tiên sử dụng MingCute icons qua các icon font/svg tối ưu, giúp giảm bundle size và đồng nhất UI theo phong cách Apple UIKit.
*   **IPC Decorators:** Trong Electron, việc giao tiếp giữa Main và Renderer được đóng gói qua `electron-ipc-decorator`, giúp code tường minh, dễ hiểu thay vì dùng các hàm gửi nhận event rời rạc.
*   **MCP (Model Context Protocol):** Tích hợp giao thức mới của Anthropic để mở rộng khả năng của AI, cho phép AI tương tác với các công cụ bên ngoài (như Fabric).

---

## 4. Tóm tắt luồng hoạt động Project (Workflow Summary)

1.  **Nạp nguồn (Subscription):**
    *   Người dùng thêm RSS, link website (qua RSSHub) hoặc import file OPML.
    *   Dữ liệu nguồn được quản lý và phân loại theo Category.
2.  **Xử lý dữ liệu (Fetching & Parsing):**
    *   App fetch dữ liệu từ API server hoặc trực tiếp từ các nguồn RSS.
    *   Bài viết được lưu vào SQLite nội bộ thông qua Drizzle ORM.
    *   Trạng thái "đã đọc" (unread count) được đồng bộ giữa local và cloud.
3.  **Tăng cường bằng AI (AI Enhancement):**
    *   Khi người dùng xem bài viết, hệ thống tự động gọi AI (nếu cấu hình) để tạo bản tóm tắt hoặc dịch nội dung.
    *   AI gán nhãn (tags) tự động để người dùng dễ dàng tìm kiếm.
4.  **Trải nghiệm người dùng (Rendering):**
    *   Người dùng có thể chọn nhiều chế độ xem: Article (truyền thống), Social Media (dạng feed), Picture (dạng gallery), Video (chơi video trực tiếp).
    *   Hỗ trợ phím tắt kiểu Vim để điều hướng cực nhanh trên Desktop.
5.  **Tương tác mở rộng (Integration):**
    *   Hỗ trợ đẩy bài viết sang các công cụ như Obsidian, Cubox, Readwise hoặc tải magnet link về qBittorrent.

---

### Kết luận
Folo là một dự án **đỉnh cao về kỹ thuật React/TypeScript**. Nó kết hợp nhuần nhuyễn giữa sự mượt mà của ứng dụng Native (qua các kỹ thuật tối ưu Electron/Expo) và sức mạnh của AI hiện đại, là một hình mẫu cho các ứng dụng "AI-First" trong tương lai.