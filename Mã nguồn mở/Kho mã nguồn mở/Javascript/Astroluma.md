Dưới đây là bản phân tích chi tiết dự án **Astroluma** dựa trên mã nguồn bạn đã cung cấp, được trình bày dưới dạng file README tiếng Việt chuyên sâu.

---

# Phân Tích Kỹ Thuật Dự Án Astroluma

Astroluma là một Dashboard quản trị Home Lab tự lưu trữ (self-hosted), được xây dựng theo mô hình hiện đại, chú trọng vào tính tùy biến và khả năng tích hợp mạnh mẽ.

## 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án sử dụng stack **MERN** (MongoDB, Express, React, Node.js) kết hợp với các công cụ tối ưu hóa hiệu suất:

### Frontend (Client)
*   **React 18 & Vite**: Sử dụng Vite làm Build Tool thay vì CRA truyền thống để tăng tốc độ phát triển và tối ưu hóa Bundle size.
*   **Recoil**: Thư viện quản lý State tập trung (State Management), thay thế cho Redux nhờ sự gọn nhẹ và khả năng tương thích tốt với các tính năng mới của React (Concurrent Mode).
*   **Tailwind CSS**: Framework CSS theo hướng tiện ích (Utility-first), kết hợp với `tw-colors` để xử lý hệ thống 15+ theme màu khác nhau.
*   **Framer Motion**: Thư viện xử lý hiệu ứng chuyển động (Animation), tạo trải nghiệm người dùng mượt mà khi tương tác với các widget.
*   **Dnd-kit**: Bộ công cụ kéo thả (Drag and Drop) mạnh mẽ được sử dụng để người dùng tự sắp xếp bố cục Dashboard.
*   **Mpegts.js**: Thư viện xử lý luồng stream video (RTSP) từ IP Camera ngay trên trình duyệt.

### Backend (Server)
*   **Node.js & Express**: Nền tảng server-side chính, xử lý các RESTful API.
*   **MongoDB & Mongoose**: Cơ sở dữ liệu NoSQL linh hoạt, phù hợp để lưu trữ cấu hình Dashboard đa dạng của nhiều người dùng khác nhau.
*   **WebSocket (ws)**: Sử dụng để truyền tải dữ liệu thời gian thực, đặc biệt là duy trì kết nối cho các luồng stream video và giám sát thiết bị mạng.
*   **Sharp**: Thư viện xử lý hình ảnh phía server (resize, convert icon).

### Hạ tầng & Triển khai
*   **Docker & Docker Compose**: Đóng gói ứng dụng thành các Container để dễ dàng triển khai trên mọi môi trường.
*   **Host Networking Mode**: Sử dụng chế độ mạng `host` để hỗ trợ quét các thiết bị trong mạng LAN (ARP Scan) và Wake-on-LAN (WOL).

---

## 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án được thiết kế với tư duy **Modularization (Mô-đun hóa)** và **Multi-tenancy (Đa người dùng)**:

*   **Kiến trúc Client-Server tách biệt**: Frontend (Vite) và Backend (Express) giao tiếp hoàn toàn qua API, cho phép dễ dàng nâng cấp hoặc thay thế từng phần.
*   **Hệ thống Integration dựa trên Manifest**: Các ứng dụng bên thứ ba (Sonarr, Portainer, Proxmox...) được tích hợp thông qua các file `manifest.json` và script riêng. Điều này cho phép mở rộng tính năng mà không cần can thiệp vào logic cốt lõi của Dashboard.
*   **State Persistance (Lưu trữ trạng thái)**: Sử dụng `recoil-persist` để lưu các tùy chỉnh của người dùng (Theme, Sidebar, cài đặt cá nhân) trực tiếp vào `localStorage`, giúp Dashboard giữ nguyên trạng thái sau khi tải lại trang.
*   **Atomic Design**: Các UI Component được chia nhỏ (NiceInput, NiceButton, NiceModal...) giúp tăng khả năng tái sử dụng và đảm bảo tính nhất quán về thiết kế.

---

## 3. Các Kỹ Thuật Chính (Key Techniques)

### Hệ thống Quản lý Icon thông minh
*   Astroluma hỗ trợ nhiều bộ Icon Pack tùy chỉnh. Người dùng có thể import các bộ icon qua URL JSON. Hệ thống hỗ trợ cả chế độ Light/Dark cho từng icon cụ thể.

### Xử lý Stream Video độ trễ thấp
*   Dự án sử dụng `mpegts.js` để giải mã luồng stream video trực tiếp từ Camera IP (thường là RTSP/RTMP) sang định dạng mà trình duyệt có thể phát được mà không cần các plugin bổ sung.

### Quản lý Bảo mật & TOTP
*   Tích hợp sẵn bộ tạo mã xác thực hai lớp (TOTP) giống như Google Authenticator ngay trên Dashboard. Kỹ thuật mã hóa AES-256 được sử dụng để bảo vệ các Secret Key trong cơ sở dữ liệu.

### Giám sát Mạng & WOL
*   Sử dụng `arp-scan` và `ping` để theo dõi trạng thái online/offline của các thiết bị trong nhà. Tính năng Wake-on-LAN gửi gói tin Magic Packet để khởi động máy tính từ xa qua giao diện web.

### Dynamic Form Generation
*   Hệ thống tự động tạo các form cấu hình (AppConfigurator.jsx) dựa trên file định nghĩa cấu trúc của ứng dụng tích hợp. Người dùng chỉ cần điền thông tin, hệ thống sẽ tự validate và lưu trữ.

---

## 4. Luồng Hoạt Động Của Hệ Thống (Workflow)

1.  **Khởi tạo (Bootstrap)**:
    *   Docker Compose khởi chạy container Node.js và MongoDB.
    *   Server chạy script `setup` để kiểm tra các phụ thuộc của ứng dụng tích hợp.
2.  **Xác thực (Authentication)**:
    *   Người dùng đăng nhập -> Server cấp JWT Token.
    *   `PrivateRoute.jsx` ở Frontend kiểm tra Token và tải dữ liệu người dùng (`userDataState`).
3.  **Tải Dashboard (Data Fetching)**:
    *   Hệ thống gọi API `/api/v1/dashboard` để lấy toàn bộ danh sách liên kết, thiết bị mạng, và các widget tích hợp.
    *   Dữ liệu được phân phối vào các Recoil Atoms để các component con sử dụng.
4.  **Tương tác (Interaction)**:
    *   Người dùng click vào liên kết -> Hệ thống kiểm tra đó là link ngoài hay trang nội bộ (Custom Page).
    *   Người dùng sắp xếp lại Dashboard -> `dnd-kit` ghi lại vị trí mới -> Gửi API cập nhật thứ tự (`reorder`) về Server.
5.  **Giám sát & Thông báo (Monitoring)**:
    *   Các widget tích hợp (ví dụ: Proxmox/Portainer) định kỳ cập nhật dữ liệu thông qua Server.
    *   Server xử lý và trả về kết quả định dạng qua file Template (`.tpl`) để Frontend hiển thị.

---
*Phân tích bởi AI dựa trên mã nguồn Astroluma v1.0.2.*