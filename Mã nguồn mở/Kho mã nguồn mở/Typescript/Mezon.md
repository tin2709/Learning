Dựa trên cấu trúc thư mục Nx Monorepo và nội dung mã nguồn của dự án **Mezon**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của nền tảng này:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Nx Monorepo:** Sử dụng Nx để quản lý nhiều ứng dụng (`apps/`) và thư viện dùng chung (`libs/`) trong một kho lưu trữ duy nhất, giúp tối ưu hóa việc tái sử dụng code và tốc độ build.
*   **Frontend Ecosystem:**
    *   **React 18 & TypeScript:** Framework chính cho Web và Admin dashboard.
    *   **Tailwind CSS & SCSS:** Xử lý giao diện và thiết kế hệ thống (Design System).
    *   **Redux Toolkit (RTK) & RTK Query:** Quản lý trạng thái toàn cục (State Management) và gọi API/Caching hiệu quả.
*   **Cross-platform:**
    *   **Electron:** Đóng gói ứng dụng Desktop (Windows, macOS, Linux).
    *   **React Native:** Phát triển ứng dụng Mobile (iOS/Android).
*   **Real-time & Media:**
    *   **WebSocket & Binary Protocol:** Sử dụng giao thức nhị phân (có dấu hiệu của Protobuf qua file `.d.ts`) để truyền tải tin nhắn với độ trễ cực thấp.
    *   **WebRTC (Pion/Livekit):** Xử lý voice chat, video call và streaming hỗ trợ hàng nghìn người dùng.
*   **Infrastructure (Backend/Storage):**
    *   ScyllaDB (NoSQL hiệu năng cao), Redis (Caching), Minio (Lưu trữ file), imgproxy (Xử lý ảnh).

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc hướng Thư viện (Lib-Oriented Architecture):**
    *   Mezon không viết logic trực tiếp trong ứng dụng. Hầu hết logic nghiệp vụ nằm ở `libs/`.
    *   `libs/store`: Chứa toàn bộ Redux slices (logic xử lý dữ liệu).
    *   `libs/transport`: Xử lý tầng mạng, kết nối Socket và Minio.
    *   `libs/ui`: Bộ UI kit dùng chung cho cả Web, Desktop và Mobile.
*   **Zero-Knowledge & Security-First:**
    *   Thiết kế tập trung vào mã hóa đầu cuối (E2EE) và bảo vệ XSS ngay từ tầng render tin nhắn.
    *   Tư duy "Zero-knowledge" đảm bảo quyền riêng tư tối đa cho người dùng.
*   **Khả năng mở rộng (Extensibility):**
    *   Hệ thống được thiết kế theo hướng "Bot-first" và "Integration-first". Có các SDK riêng cho nhiều ngôn ngữ (Go, Java, Python, NestJS) để bên thứ ba phát triển ứng dụng trên nền tảng.
*   **Tối ưu hóa hiệu năng (Performance Optimization):**
    *   Sử dụng `Fasterdom` (trong `libs/utils`) để kiểm soát việc đọc/ghi vào DOM, giảm thiểu hiện tượng giật lag (Reflow/Repaint) khi có hàng nghìn tin nhắn.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Binary Protocol qua WebSockets:** Thay vì gửi tin nhắn JSON thuần, Mezon sử dụng Protobuf hoặc định dạng nhị phân tương đương để nén dữ liệu, giúp việc gửi tin nhắn cực nhanh và tiết kiệm băng thông.
*   **Virtual Scrolling & Idle Rendering:** Kỹ thuật xử lý danh sách tin nhắn cực dài trong `libs/chat-scroll` để đảm bảo trình duyệt không bị treo.
*   **Web Workers & Offscreen Canvas:** Sử dụng Worker để xử lý các tác vụ nặng như làm mờ ảnh (Blurhash) hoặc xử lý media mà không gây block luồng chính của UI.
*   **E2EE Implementation:** Quản lý Keystore và mã hóa dữ liệu ngay tại client trước khi gửi lên server qua tầng `libs/utils/e2ee`.
*   **Micro-Frontend mindset:** Các ứng dụng nhỏ như `discover`, `admin`, `chat` được chia tách rõ ràng nhưng chia sẻ chung tầng `core` và `store`.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi tạo (Bootstrap):** Khi ứng dụng chạy (Web hoặc Desktop), `main.tsx` sẽ kích hoạt `AppLoader`. Tầng này kiểm tra phiên bản, cấu hình giao diện (Theme) và thiết lập ngôn ngữ (i18n).
2.  **Xác thực (Authentication):** `AuthLoader` kiểm tra Token trong LocalStorage/Session. Dữ liệu người dùng được lưu vào `auth.slice.ts` trong Redux Store.
3.  **Kết nối Real-time:**
    *   Sau khi login, `MezonContext` thiết lập kết nối WebSocket thông qua `libs/transport`.
    *   Socket lắng nghe các sự kiện (tin nhắn mới, người dùng online, đổi vai trò).
4.  **Xử lý dữ liệu (Data Flow):**
    *   Tin nhắn nhị phân đến -> Giải mã qua Protobuf -> Đưa vào Redux Store (`messages.slice.ts`).
    *   UI (React components) lắng nghe Store và tự động re-render.
5.  **Tương tác Media:** Khi người dùng vào Voice Channel, Livekit/WebRTC được kích hoạt thông qua `useWebRTCCall` hook, thiết lập luồng âm thanh/hình ảnh trực tiếp giữa các client hoặc qua SFU server.
6.  **Lưu trữ file:** Khi gửi file, ứng dụng gọi trực tiếp đến Minio (S3 compatible) qua `libs/transport/minio` và gửi URL/Metadata trở lại kênh chat qua Socket.

**Tổng kết:** Mezon là một dự án có kỹ thuật rất cao, áp dụng các tiêu chuẩn công nghệ hiện đại nhất của năm 2024-2026 để giải quyết bài toán giao tiếp thời gian thực quy mô lớn, tương đương với các nền tảng hàng đầu thế giới như Discord hay Slack.