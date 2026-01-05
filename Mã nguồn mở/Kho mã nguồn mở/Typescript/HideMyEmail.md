Dưới đây là bản phân tích chuyên sâu về kỹ thuật và kiến trúc của dự án **Hide My Email (Cloudflare)**, được trình bày dưới dạng một file README tiếng Việt chuyên nghiệp.

---

# Phân Tích Kỹ Thuật: Hide My Email (with Cloudflare)

Dự án này là một tiện ích mở rộng trình duyệt (Browser Extension) hiện đại, được xây dựng với tư duy hệ thống cao, sử dụng kiến trúc Monorepo để quản lý quy mô và tính tái sử dụng của mã nguồn.

## 1. Công Nghệ Cốt Lõi (Core Technology Stack)

*   **Framework:** [React 19](https://react.dev/) - Sử dụng phiên bản mới nhất để tận dụng hiệu năng và các hook tối ưu.
*   **Ngôn ngữ:** [TypeScript](https://www.typescriptlang.org/) (95%+) - Đảm bảo tính an toàn về kiểu dữ liệu trên toàn bộ hệ thống.
*   **Build Tool:** [Vite](https://vitejs.dev/) - Thay thế cho Webpack truyền thống để tăng tốc độ phát triển và đóng gói.
*   **Quản lý Monorepo:** [PNPM Workspaces](https://pnpm.io/workspaces) & [Turborepo](https://turbo.build/) - Tối ưu hóa việc cài đặt dependency và song song hóa các tác vụ build/lint.
*   **Styling:** [Tailwind CSS](https://tailwindcss.com/) - Quản lý giao diện nhất quán thông qua các config dùng chung.
*   **Cloudflare Integration:** [Cloudflare SDK](https://github.com/cloudflare/cloudflare-nodejs) - Tương tác trực tiếp với API của Cloudflare để cấu hình Email Routing.

## 2. Kỹ Thuật & Tư Duy Kiến Trúc (Architecture Thinking)

Dự án được thiết kế theo mô hình **Modular Monorepo**, chia nhỏ hệ thống thành các gói (packages) độc lập:

*   **Tính đóng gói (Encapsulation):** Các logic quan trọng như `storage`, `i18n`, `ui` được tách ra thành các package riêng biệt trong thư mục `packages/`. Điều này giúp mã nguồn ở `pages/popup` rất sạch sẽ, chỉ tập trung vào logic hiển thị.
*   **Trình thông dịch Manifest (Manifest Parser):** Dự án sử dụng một script TypeScript (`chrome-extension/utils/plugins/make-manifest-plugin.ts`) để tự động chuyển đổi manifest giữa Chrome (v3) và Firefox, xử lý các khác biệt về `background service worker` và `permissions`.
*   **Giao tiếp dữ liệu (Reactive Storage):** Thay vì gọi trực tiếp `chrome.storage`, dự án xây dựng một lớp trừu tượng `BaseStorage` kết hợp với `useSyncExternalStore` của React, giúp UI tự động cập nhật ngay khi dữ liệu trong storage thay đổi.

## 3. Các Kỹ Thuật Key (Key Engineering Techniques)

### A. Chiến lược "Pre-created Rules" (Tối ưu Cloudflare API)
Một trong những kỹ thuật thông minh nhất của dự án là việc tạo sẵn các quy tắc email trống (tối đa 180-200 rules) trong quá trình cài đặt ban đầu.
*   **Lý do:** API Cloudflare có độ trễ khi đồng bộ settings. Việc tạo sẵn giúp người dùng có trải nghiệm "tức thì" khi cần một email mới, vì extension chỉ cần cập nhật nhãn (label) của một rule đã tồn tại thay vì tạo mới hoàn toàn.

### B. Custom HMR (Hot Module Replacement) cho Extension
Phát triển Extension thường gặp khó khăn do phải reload thủ công. Dự án triển khai gói `@extension/hmr`:
*   Sử dụng **WebSocket server** để theo dõi thay đổi code.
*   Tự động gửi tín hiệu để reload background script và refresh lại giao diện popup ngay khi lưu file.

### C. Internationalization (i18n) Type-safe
Gói `@extension/i18n` không chỉ đơn thuần là dịch thuật:
*   Nó kiểm tra kiểu dữ liệu của các key trong file `messages.json`. Nếu bạn dùng một key chưa định nghĩa, TypeScript sẽ báo lỗi ngay lập tức.

### D. Module Manager CLI
Dự án cung cấp một công cụ dòng lệnh nội bộ (`packages/module-manager`) cho phép nhà phát triển:
*   Xóa hoặc khôi phục các tính năng (Popup, Side Panel, Content Script) một cách nhanh chóng thông qua giao diện tương tác (Inquirer.js).

## 4. Tóm Tắt Luồng Hoạt Động (Project Workflow)

### Bước 1: Thiết lập (Setup Flow)
1.  Người dùng nhập **API Token**, **Zone ID** và **Account ID** từ Cloudflare.
2.  Extension kiểm tra quyền truy cập API.
3.  **Hành động nền:** Extension thực hiện tạo hàng loạt (batch) các Email Routing rules với định danh `[hide_mail]` và trạng thái `unused`.

### Bước 2: Tạo Email mới (Email Generation Flow)
1.  Người dùng nhấn "Create New".
2.  Hệ thống tìm trong danh sách "Pre-created Rules" một rule đang ở trạng thái `unused`.
3.  Người dùng nhập nhãn (ví dụ: "Đăng ký Netflix").
4.  Extension gọi API Cloudflare để cập nhật tên rule và kích hoạt nó.
5.  Email ngẫu nhiên (ví dụ: `red-panda-42@yourdomain.com`) sẵn sàng sử dụng.

### Bước 3: Quản lý & Đồng bộ (Management Flow)
1.  Mọi dữ liệu về danh sách email được lấy trực tiếp từ Cloudflare thông qua API để đảm bảo tính đồng bộ trên nhiều thiết bị.
2.  Khi người dùng xóa email, extension thực chất sẽ "reset" rule đó về trạng thái `unused` để tái sử dụng sau này.

---

## 5. Kết Luận
Đây là một dự án mẫu mực cho việc áp dụng **Modern Frontend Tooling** vào phát triển Browser Extension. Nó giải quyết tốt bài toán về hiệu suất API (thông qua pre-creation) và trải nghiệm nhà phát triển (thông qua Monorepo và Custom HMR).