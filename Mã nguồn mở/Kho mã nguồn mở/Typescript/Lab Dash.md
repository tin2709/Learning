Dựa trên mã nguồn và cấu trúc tệp tin của dự án **Lab Dash**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng ngăn xếp công nghệ (stack) hiện đại, tập trung vào tính hiệu quả và khả năng đóng gói:

*   **Ngôn ngữ:** **TypeScript (99%+)** được sử dụng xuyên suốt cả Backend và Frontend, giúp đảm bảo tính an toàn về kiểu dữ liệu (type-safety) và dễ bảo trì.
*   **Frontend:**
    *   **React 18:** Thư viện giao diện chính.
    *   **Vite:** Công cụ build siêu tốc thay thế cho CRA truyền thống.
    *   **Material UI (MUI) 6:** Framework UI để thiết kế dashboard chuyên nghiệp, hỗ trợ Responsive tốt.
    *   **@dnd-kit:** Bộ thư viện xử lý kéo thả (Drag and Drop) phức tạp cho các widget.
*   **Backend:**
    *   **Node.js & Express:** Framework server-side.
    *   **Systeminformation:** Thư viện quan trọng để lấy thông số phần cứng (CPU, RAM, Disk, Network).
    *   **JSON Storage:** Thay vì sử dụng cơ sở dữ liệu nặng nề (SQL/NoSQL), dự án lưu trữ cấu hình trực tiếp vào các tệp `.json` (config.json, users.json), tối ưu cho môi trường homelab.
*   **Hạ tầng & Triển khai:**
    *   **Docker & Docker Compose:** Đóng gói toàn bộ ứng dụng vào container.
    *   **Kubernetes (Helm Chart):** Hỗ trợ triển khai trên các cụm máy chủ lớn.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Lab Dash đi theo hướng **"Simple yet Robust"** (Đơn giản nhưng mạnh mẽ):

*   **Kiến trúc Hướng dữ liệu (Data-Driven UI):** Toàn bộ Dashboard được hình thành từ một tệp cấu hình JSON. Khi người dùng thay đổi vị trí widget hoặc thêm dịch vụ, hệ thống chỉ cập nhật JSON và giao diện sẽ tự động Render lại.
*   **Backend làm Proxy (Proxy Gateway):** Để tránh lỗi CORS khi gọi API từ trình duyệt đến các dịch vụ như Pi-hole, AdGuard hay qBittorrent, Backend đóng vai trò là "người trung gian". Frontend gửi yêu cầu đến Backend, Backend thực hiện gọi dịch vụ và trả kết quả về.
*   **Bảo mật "Local-First":**
    *   Dữ liệu nhạy cảm (API Key, mật khẩu dịch vụ) được mã hóa bằng **AES-256-CBC** ngay tại Backend bằng khóa bí mật (`SECRET`) do người dùng tự tạo.
    *   Sử dụng **JWT (JSON Web Token)** lưu trong **HttpOnly Cookie** để quản lý phiên đăng nhập, giúp chống lại các cuộc tấn công XSS.
*   **Thiết kế Mobile-First & Responsive:** Dashboard tách biệt cấu hình layout cho Desktop và Mobile trong cùng một file config, cho phép người dùng tùy chỉnh trải nghiệm trên từng loại thiết bị.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Xử lý Kéo thả phức tạp:** Sử dụng `DndContext`, `SortableContext` kết hợp với các giải thuật phát hiện va chạm (collision detection) tùy chỉnh để xử lý việc kéo thả widget vào các Group hoặc sắp xếp lại vị trí trong Grid.
*   **Quản lý Trạng thái (State Management):** Sử dụng **React Context API** (`AppContextProvider`) thay vì Redux. Điều này giúp giảm độ phức tạp nhưng vẫn quản lý tốt các trạng thái toàn cục như cấu hình Dashboard, trạng thái đăng nhập và thông tin người dùng.
*   **Tối ưu hóa hiệu năng (Performance Optimization):**
    *   **Bulk Loading:** Có các Route API riêng (`/api/icons/bulk`, `/api/widgets/bulk-data`) để tải hàng loạt dữ liệu ban đầu thay vì gọi hàng chục API nhỏ lẻ, giúp giảm độ trễ khi tải trang.
    *   **Caching:** Backend có cơ chế Cache in-memory cho các icon để giảm việc đọc file liên tục từ đĩa cứng.
*   **Mã hóa & Bảo mật:**
    *   Kỹ thuật **Masking:** Khi gửi cấu hình về Frontend, các thông tin nhạy cảm như mật khẩu sẽ bị xóa bỏ hoặc thay bằng dấu sao (`**********`). Backend chỉ trả về cờ (flag) như `_hasPassword: true`.
    *   Sử dụng `bcrypt` để băm (hash) mật khẩu người dùng admin.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng hoạt động của ứng dụng diễn ra như sau:

1.  **Giai đoạn Khởi tạo:**
    *   Người dùng chạy Container. Backend kiểm tra `users.json`. Nếu trống, hệ thống kích hoạt **Setup Wizard** để người dùng tạo tài khoản Admin đầu tiên.
2.  **Giai đoạn Tải dữ liệu:**
    *   Frontend gọi API `/api/config` để lấy cấu hình Dashboard.
    *   Dựa trên ID dịch vụ trong cấu hình, Frontend gọi các API proxy (ví dụ: `/api/system`, `/api/weather`, `/api/qbittorrent/stats`).
    *   Backend nhận yêu cầu, giải mã thông tin đăng nhập từ `config.json`, thực hiện gọi dịch vụ bên ngoài và trả dữ liệu sạch về cho UI.
3.  **Giai đoạn Tương tác:**
    *   Khi người dùng bật **Edit Mode**, họ có thể kéo thả widget.
    *   Mỗi thay đổi vị trí sẽ kích hoạt một sự kiện cập nhật State cục bộ trong React để đảm bảo độ mượt (Optimistic UI).
    *   Sau đó, một yêu cầu POST được gửi đến `/api/config` để lưu trạng thái mới vào file JSON trên server.
4.  **Giai đoạn Giám sát:**
    *   Các widget (như System Monitor) thiết lập cơ chế **Polling** (gọi lại API sau mỗi 2-5 giây) để cập nhật thông số thời gian thực mà không cần tải lại toàn bộ trang.

**Tổng kết:** Lab Dash là một dự án homelab điển hình có thiết kế rất tinh gọn, thực dụng, tập trung mạnh vào khả năng tùy biến của người dùng cuối trong khi vẫn đảm bảo các tiêu chuẩn bảo mật cho môi trường tự triển khai (self-hosted).