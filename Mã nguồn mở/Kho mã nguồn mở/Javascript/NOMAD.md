Dựa trên mã nguồn và cấu trúc thư mục của dự án **NOMAD** (Navigation Organizer for Maps, Activities & Destinations), dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng mô hình **Fullstack JavaScript** hiện đại, tối ưu cho việc tự triển khai (self-hosted) và hiệu năng nhẹ nhàng.

*   **Backend:** 
    *   **Node.js 22 & Express:** Framework nền tảng xử lý logic và API.
    *   **SQLite (`better-sqlite3`):** Lựa chọn thông minh cho ứng dụng self-hosted. SQLite là cơ sở dữ liệu dạng file, không cần cài đặt server DB phức tạp, nhưng vẫn đảm bảo tốc độ cao nhờ thư viện `better-sqlite3`.
    *   **WebSocket (`ws`):** Cung cấp khả năng giao tiếp hai chiều thời gian thực.
*   **Frontend:**
    *   **React 18 & Vite:** Đảm bảo giao diện phản hồi nhanh và quá trình phát triển tối ưu.
    *   **Zustand:** Thư viện quản lý State cực kỳ nhẹ (thay thế Redux), giúp đồng bộ dữ liệu giữa các thành phần giao diện một cách đơn giản.
    *   **Tailwind CSS:** Xử lý giao diện (UI) linh hoạt, hỗ trợ tốt Dark Mode và Responsive.
*   **Tính năng PWA (Progressive Web App):**
    *   **Workbox:** Quản lý Service Worker để lưu bộ nhớ đệm (cache) bản đồ, dữ liệu API, giúp ứng dụng hoạt động mượt mà ngay cả khi offline hoặc mạng yếu khi đang đi du lịch.
*   **Bản đồ & Thời tiết:**
    *   **Leaflet:** Thư viện bản đồ mã nguồn mở chính.
    *   **OpenStreetMap & Google Places API:** Kết hợp giữa tìm kiếm miễn phí và dữ liệu địa điểm phong phú.
    *   **Open-Meteo API:** Dự báo thời tiết không cần API key.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của NOMAD tập trung vào sự **tinh gọn** và **khả năng cộng tác**.

*   **Kiến trúc hướng cộng tác (Real-time Collaboration):** Hệ thống không chỉ lưu dữ liệu vào DB mà còn coi WebSocket là "trục xương sống". Khi một người thay đổi lịch trình, tất cả những người khác đang xem cùng một chuyến đi sẽ thấy thay đổi ngay lập tức.
*   **Thiết kế Mobile-First:** Vì là ứng dụng du lịch, giao diện được tối ưu hóa cho cảm ứng, hỗ trợ cài đặt trực tiếp lên điện thoại (PWA) mà không cần qua App Store/Play Store.
*   **Tư duy Modular (Addons):** Các tính năng nâng cao như `Vacay` (quản lý ngày nghỉ) hay `Atlas` (bản đồ thế giới cá nhân) được thiết kế dạng module, có thể bật/tắt bởi quản trị viên.
*   **Dữ liệu tập trung vào File-based:** Cả Database (SQLite) và các tệp tải lên (Uploads) đều lưu trữ trong các thư mục định sẵn, giúp việc Backup và di chuyển dữ liệu giữa các server trở nên cực kỳ dễ dàng (chỉ cần copy thư mục).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Optimistic Updates (Cập nhật lạc quan):** Ở Frontend, khi người dùng thay đổi dữ liệu (ví dụ: kéo thả một địa điểm), giao diện sẽ cập nhật ngay lập tức trước khi nhận được phản hồi từ server. Nếu server lỗi, trạng thái sẽ được hoàn tác (rollback).
*   **Zustand Store Sharding:** Chia nhỏ Store quản lý trạng thái theo từng domain (`authStore`, `tripStore`, `settingsStore`), giúp code sạch sẽ và dễ bảo trì.
*   **JWT & OIDC Authentication:** Sử dụng JSON Web Token để xác thực API và hỗ trợ OpenID Connect để tích hợp với các hệ thống đăng nhập một lần (SSO) như Google, Apple, Keycloak.
*   **Automatic Backup & Scheduler:** Tích hợp sẵn một bộ lập lịch (`scheduler.js`) trong server để tự động sao lưu toàn bộ hệ thống theo chu kỳ, đảm bảo an toàn dữ liệu cho người dùng.
*   **PDF Rendering:** Sử dụng `@react-pdf/renderer` để tạo ra các bản kế hoạch du lịch chuyên nghiệp dưới dạng PDF ngay trên trình duyệt.

### 4. Luồng hoạt động hệ thống (System Flow)

#### A. Luồng Đăng nhập và Xác thực:
1. Người dùng gửi thông tin qua `authApi`.
2. Server xác thực và trả về **JWT Token**.
3. Frontend lưu Token vào `localStorage` và cập nhật `authStore`.
4. Các yêu cầu API sau đó sẽ tự động đính kèm Token này qua `axios interceptors`.

#### B. Luồng Cộng tác thời gian thực (Real-time Sync):
1. Người dùng mở một chuyến đi (`tripId`).
2. Frontend gửi yêu cầu `join` qua WebSocket đến server.
3. Server đưa socket đó vào một "phòng" (Room) đại diện cho `tripId`.
4. Khi người dùng A thay đổi lịch trình:
    *   API gửi yêu cầu cập nhật đến Server.
    *   Server cập nhật SQLite.
    *   Server phát (Broadcast) một thông điệp qua WebSocket đến tất cả các socket trong cùng "phòng".
    *   Các thành viên khác nhận thông điệp và tự động kích hoạt hàm refetch dữ liệu hoặc cập nhật Store cục bộ.

#### C. Luồng tìm kiếm và lưu địa điểm:
1. Người dùng nhập tên địa điểm.
2. `mapsApi` gọi đến proxy server (để bảo mật API Key).
3. Server gọi Google Places hoặc OSM và trả kết quả về.
4. Khi lưu, tọa độ (lat, lng) được lưu vào SQLite để hiển thị lên bản đồ Leaflet.

### Tổng kết
**NOMAD** là một ví dụ điển hình của việc sử dụng các công nghệ JavaScript nhẹ nhàng nhưng mạnh mẽ để tạo ra một ứng dụng có trải nghiệm người dùng cao (UX). Nó ưu tiên sự đơn giản trong triển khai (SQLite, Docker) nhưng không đánh đổi các tính năng hiện đại như Real-time và PWA.