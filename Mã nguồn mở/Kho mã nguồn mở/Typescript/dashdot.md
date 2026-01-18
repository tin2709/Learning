Dưới đây là bản phân tích chi tiết và tóm tắt dự án **dash. (dashdot)** dưới dạng một file README tiếng Việt chuyên sâu, tập trung vào công nghệ và kiến trúc.

---

# dash. (dashdot) - Phân Tích Kiến Trúc & Công Nghệ

**dash.** là một bảng điều khiển (dashboard) máy chủ hiện đại, tập trung vào tính thẩm mỹ (glassmorphism) và hiệu suất cao cho các máy chủ VPS nhỏ hoặc máy chủ cá nhân.

## 1. Công Nghệ Cốt Lõi (Tech Stack)

Dự án được xây dựng theo mô hình **Monorepo** sử dụng **Yarn Workspaces** và **Turbo (Turborepo)** để quản lý các gói ứng dụng:

### Frontend (apps/view)
*   **React 19 & TypeScript**: Thư viện UI chính.
*   **Vite**: Build tool thế hệ mới cho tốc độ phản hồi cực nhanh.
*   **Styled-components**: Quản lý CSS-in-JS, giúp hiện thực hóa phong cách thiết kế **Glassmorphism**.
*   **Recharts**: Thư viện biểu đồ mạnh mẽ để trực quan hóa dữ liệu CPU, RAM, Network.
*   **Framer Motion**: Xử lý các hiệu ứng chuyển động mượt mà.
*   **Socket.io-client**: Kết nối thời gian thực với backend.

### Backend (apps/server)
*   **Node.js & Express**: Máy chủ API và phục vụ tài nguyên tĩnh.
*   **RxJS**: Trái tim của hệ thống xử lý luồng dữ liệu (Streams). Sử dụng để quản lý polling dữ liệu phần cứng và phát tới client.
*   **Socket.io**: Giao thức truyền tin thời gian thực giữa server và client.
*   **systeminformation**: Thư viện quan trọng nhất để thu thập thông số phần cứng từ hệ điều hành (CPU, RAM, Disks, GPU, Network).
*   **node-cron**: Quản lý các tác vụ định kỳ như kiểm tra tốc độ mạng (Speedtest).

### Infrastructure & Tools
*   **Docker & Docker Compose**: Đóng gói ứng dụng, hỗ trợ cả kiến trúc x64 và ARM (Raspberry Pi). Đặc biệt có bản riêng cho **NVIDIA GPU**.
*   **Docusaurus**: Xây dựng trang tài liệu kỹ thuật chuyên nghiệp.
*   **Biome**: Công cụ thay thế ESLint/Prettier để lint và format code với tốc độ cao.

## 2. Tư Duy Kiến Trúc (Architectural Thinking)

### Mô hình Monorepo
Dự án chia làm các khối logic rõ ràng:
*   `apps/server`: Logic thu thập dữ liệu và API.
*   `apps/view`: Giao diện người dùng.
*   `apps/cli`: Công cụ dòng lệnh để debug dữ liệu thô.
*   `packages/common`: Chứa các `Types` (TypeScript) dùng chung cho cả frontend và backend, đảm bảo tính nhất quán của dữ liệu.

### Real-time Data Streaming
Thay vì sử dụng HTTP Request liên tục (polling từ client), server sử dụng **RxJS Observables** để tạo ra các luồng dữ liệu từ phần cứng. Khi một client kết nối qua Socket.io, server sẽ "pipe" luồng dữ liệu này trực tiếp đến client đó. Điều này giảm thiểu overhead và độ trễ.

### Chiến lược "Mount Host"
Vì chạy trong Docker, để đọc được thông số của máy chủ thật (host), dự án sử dụng kỹ thuật mount thư mục gốc `/` của host vào `/mnt/host` trong container với quyền chỉ đọc (`ro`). Backend sẽ đọc các file hệ thống như `/etc/os-release` hoặc `/proc` thông qua đường dẫn này.

## 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Glassmorphism UI**: Sử dụng `backdrop-filter: blur()` và màu sắc có độ trong suốt cao (alpha channel) trong `Styled-components` để tạo hiệu ứng kính mờ đặc trưng.
*   **Dynamic Storage Mapping**: Kỹ thuật ánh xạ thông minh giữa các thiết bị khối (block devices) và kích thước tệp hệ thống (file system size) để hiển thị chính xác dung lượng ổ cứng, kể cả trong các cấu hình RAID phức tạp.
*   **Speedtest Integration**: Hỗ trợ cả `Ookla Speedtest CLI` và `speedtest-cli` (Python) hoặc đọc kết quả từ một file JSON tùy chỉnh.
*   **NVIDIA GPU Support**: Sử dụng Docker image dựa trên Ubuntu (thay vì Alpine) để cài đặt các driver và thư viện cần thiết nhằm giao tiếp với NVIDIA Management Library (NVML).

## 4. Luồng Hoạt Động Của Project (Project Flow)

1.  **Khởi tạo (Startup)**:
    *   Server kiểm tra môi trường (Docker hay chạy trực tiếp).
    *   Thiết lập kết nối với các file hệ thống của host (nếu trong Docker).
    *   Khởi chạy các luồng RxJS để bắt đầu poll dữ liệu phần cứng theo khoảng thời gian cấu hình (mặc định 1 giây cho CPU/RAM).

2.  **Thu thập dữ liệu (Data Collection)**:
    *   `systeminformation` truy vấn thông số từ OS.
    *   Dữ liệu được chuyển qua các bộ Mapper (ví dụ: `DynamicStorageMapper`) để chuẩn hóa định dạng theo chuẩn của gói `@dashdot/common`.

3.  **Truyền tải (Transmission)**:
    *   Dữ liệu tĩnh (Tên CPU, phiên bản OS) được gửi ngay khi client kết nối qua sự kiện `static-info`.
    *   Dữ liệu động (Tải CPU, dung lượng RAM trống) được push liên tục qua Socket.io.

4.  **Hiển thị (Visualization)**:
    *   Frontend nhận dữ liệu, lưu vào state.
    *   `Recharts` thực hiện render biểu đồ dạng Area hoặc Pie dựa trên dữ liệu mới nhất.
    *   Hệ thống kiểm tra `query-params` (ví dụ: `?graph=cpu`) để hiển thị chế độ widget đơn lẻ (thường dùng để nhúng vào trang web khác qua iframe).

## 5. Các Tính Năng Nổi Bật
*   **Đa nền tảng**: Hỗ trợ Linux, Windows và ARM.
*   **Tùy biến cao**: Cho phép ẩn/hiện widget, thay đổi màu sắc, độ bo góc qua biến môi trường hoặc tham số URL.
*   **Tích hợp**: Có sẵn các endpoint API cho các công cụ khác như Homarr hoặc Heimdall.

---
*Dự án này là một ví dụ tuyệt vời về cách kết hợp giữa tư duy lập trình hàm (Functional Programming) với RxJS và thiết kế UI hiện đại để tạo ra một công cụ giám sát máy chủ vừa mạnh mẽ vừa đẹp mắt.*