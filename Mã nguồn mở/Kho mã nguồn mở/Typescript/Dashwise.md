Dưới đây là bản phân tích chi tiết dự án **Dashwise** dưới dạng file README bằng tiếng Việt, tập trung vào công nghệ, kiến trúc và quy trình vận hành.

---

# Phân Tích Kỹ Thuật Dự Án Dashwise

**Dashwise** là một nền tảng Dashboard cá nhân (Homelab Dashboard) hiện đại, cho phép quản lý các dịch vụ tự lưu trữ (self-hosted), theo dõi tình trạng hệ thống, đọc tin tức RSS và quản lý thông báo tập trung.

## 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án sử dụng bộ Stack hiện đại, tối ưu cho hiệu năng và khả năng mở rộng:

*   **Frontend & API Layer:**
    *   **Next.js 15+ (App Router):** Sử dụng Server Components cho hiệu năng và Client Components cho tương tác người dùng.
    *   **React 19:** Tận dụng các tính năng mới nhất của React.
    *   **Tailwind CSS 4:** Framework CSS mới nhất cho giao diện linh hoạt.
    *   **Shadcn/UI:** Bộ thư viện thành phần giao diện (UI components) chất lượng cao.
    *   **Framer Motion:** Xử lý hiệu ứng chuyển động mượt mà.
*   **Backend (BaaS):**
    *   **Pocketbase:** Một Backend-as-a-Service mã nguồn mở (viết bằng Go) tích hợp sẵn Database (SQLite), Auth, và File Storage.
*   **Background Jobs:**
    *   **Node.js & Fastify:** Chạy một container riêng biệt để xử lý các tác vụ nền (Cron jobs).
*   **Containerization:**
    *   **Docker & Docker Compose:** Đóng gói đa dịch vụ (App, Pocketbase, Jobs).

## 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của Dashwise được thiết kế theo hướng **Decoupled Architecture** (Kiến trúc tách rời):

1.  **Tách biệt UI và Xử lý nền (UI & Worker Separation):** Các tác vụ nặng như kiểm tra tình trạng link (uptime monitoring), cào tin tức RSS, và lập chỉ mục tìm kiếm được chuyển sang container `jobs`. Điều này giúp ứng dụng web chính luôn mượt mà.
2.  **Cấu hình dựa trên người dùng (User-Centric Configuration):** Mỗi người dùng khi đăng ký sẽ có một bản ghi cấu hình JSON riêng biệt trong Pocketbase (`userConfig`). Mọi thay đổi về giao diện, widget, link đều được lưu vào JSON này.
3.  **Hệ thống Widget Module hóa:** Các Widget (Weather, Calendar, Integration) được thiết kế độc lập. Dashwise sử dụng cơ chế `dynamic import` để chỉ tải những Widget mà người dùng thực sự sử dụng trên màn hình.
4.  **Tích hợp dựa trên Client API:** Dashwise không trực tiếp kết nối DB của các ứng dụng khác mà thông qua các API Client (Beszel, Dashdot, Jellyfin, Karakeep) để lấy dữ liệu.

## 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Spotlight Search (Command Bar):** Sử dụng thư viện `cmdk` kết hợp với `Fuse.js` (hoặc logic filter tùy chỉnh) để tìm kiếm nhanh các dịch vụ và thực hiện lệnh (shortcut/bangs) giống như macOS Spotlight.
*   **Uptime Monitoring System:** 
    *   *Indexer:* Quét cấu hình người dùng để tìm các link yêu cầu theo dõi.
    *   *Runner:* Thực hiện request HTTP GET định kỳ để kiểm tra mã trạng thái (status code).
*   **Kéo thả (Drag-and-Drop):** Sử dụng `@dnd-kit` để cho phép người dùng tùy chỉnh vị trí các Widget trên Dashboard một cách trực quan.
*   **Notification Forwarding:** Sử dụng công cụ **Shoutrrr** bên trong container `jobs` để chuyển tiếp thông báo từ Dashwise đến các nền tảng khác như Discord, Slack, Telegram.
*   **Dynamic Font Loading:** Kỹ thuật tự động tạo thẻ `<style>` và inject `@font-face` vào DOM để thay đổi font chữ đồng hồ theo cấu hình người dùng mà không làm chậm trang.
*   **Wallpaper Processing:** Sử dụng thư viện `sharp` ở phía server để resize và tối ưu hóa ảnh nền (lên đến 4K) trước khi lưu trữ, giúp tiết kiệm băng thông.

## 4. Tóm Tắt Luồng Hoạt Động (Project Workflow)

### A. Luồng Đăng ký & Khởi tạo:
1. Người dùng đăng ký qua giao diện Dashwise.
2. API Next.js tạo User trong Pocketbase.
3. Dashwise tự động sao chép file `default-config.json` để tạo cấu hình khởi tạo cho người dùng đó.

### B. Luồng Hiển thị Dashboard:
1. Khi người dùng truy cập, `ConfigContext` gọi API `/api/v1/config`.
2. Dữ liệu JSON được trả về và phân phối xuống các thành phần UI.
3. Các Widget dựa trên cấu hình sẽ tự động gọi API riêng của chúng (ví dụ: lấy thời tiết từ Open-Meteo, lấy thông số từ Beszel).

### C. Luồng Xử lý Nền (Jobs container):
1. **Mỗi 10 phút:** Chạy `SearchItemIndexer` để cập nhật danh sách tìm kiếm từ các tích hợp (Jellyfin, Karakeep).
2. **Mỗi phút:** `MonitoringRunner` kiểm tra các link và ghi log trạng thái (healthy/unhealthy).
3. **Định kỳ:** `FeedBuilder` tải các tin RSS mới nhất và lưu vào Pocketbase để hiển thị trong mục News.

### D. Luồng Thông báo:
1. Một dịch vụ bên ngoài gửi POST request kèm Token đến Dashwise.
2. Dashwise lưu thông báo vào DB và đặt trạng thái là `queued`.
3. Container `jobs` quét các thông báo `queued` và gửi đi qua các Forwarder (Discord/Telegram) đã cấu hình.

## 5. Kết Luận

Dashwise không chỉ là một trang web hiển thị link đơn thuần, mà là một hệ thống quản lý Homelab toàn diện. Việc kết hợp giữa **Next.js** (tốc độ phát triển cao) và **Pocketbase** (nhẹ, dễ triển khai) cùng với kiến trúc **Worker-based** giúp dự án này trở nên cực kỳ mạnh mẽ nhưng vẫn dễ dàng tự lưu trữ (self-hosted) trên các thiết bị cấu hình thấp như Raspberry Pi.