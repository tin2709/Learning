Dựa trên nội dung file `README.md` và cấu trúc mã nguồn của dự án **AFFiNE**, dưới đây là phân tích chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Technology)
AFFiNE sử dụng một tập hợp công nghệ hiện đại, kết hợp giữa hiệu năng của Rust và tính linh hoạt của hệ sinh thái JavaScript:

*   **Ngôn ngữ lập trình:**
    *   **TypeScript (91.1%):** Ngôn ngữ chính cho toàn bộ giao diện và logic ứng dụng.
    *   **Rust (3.1%):** Được dùng để xây dựng **OctoBase** (cơ sở dữ liệu) và **y-octo** (bản triển khai CRDT tối ưu), giúp đảm bảo hiệu năng cực cao và xử lý dữ liệu an toàn.
*   **Framework Giao diện:**
    *   **React:** Thư viện chính cho UI.
    *   **Lit:** Được sử dụng trong **BlockSuite** để tạo các Web Components nhẹ và hiệu quả cho trình soạn thảo.
    *   **Jotai:** Quản lý state (trạng thái) ứng dụng một cách tối giản và linh hoạt.
*   **Nền tảng đa thiết bị:**
    *   **Electron:** Cho ứng dụng Desktop (Windows, macOS, Linux).
    *   **Capacitor & Swift/Kotlin:** Cho ứng dụng Mobile (iOS/Android).
*   **Hệ thống xây dựng & Tooling:**
    *   **Vite:** Công cụ build thế hệ mới cho frontend.
    *   **Napi-rs:** Cầu nối giúp gọi mã Rust từ Node.js.
    *   **Yarn (Berry/4.x):** Quản lý Monorepo với tốc độ cao.

### 2. Tư duy kiến trúc (Architectural Mindset)
Kiến trúc của AFFiNE xoay quanh 4 trụ cột chính:

*   **Local-first (Ưu tiên dữ liệu cục bộ):** Khác với Notion (Cloud-first), AFFiNE ưu tiên lưu trữ dữ liệu trên máy người dùng. Dữ liệu thuộc về bạn, hoạt động ngoại tuyến (offline) hoàn hảo và chỉ đồng bộ khi có kết nối.
*   **Everything is a Block (Mọi thứ là một khối):** Dựa trên framework **BlockSuite**. Một đoạn văn, một bảng, hay một hình vẽ đều là các "khối" có thể di chuyển, chuyển đổi và tương tác lẫn nhau.
*   **Hyper-merged (Siêu hợp nhất):** Xóa bỏ ranh giới giữa văn bản (Docs), bảng trắng (Whiteboard) và cơ sở dữ liệu (Database). Bạn có thể viết tài liệu trên một không gian vô hạn (Edgeless canvas) hoặc xem chúng dưới dạng bảng chỉ bằng một cú click.
*   **CRDT-based Collaboration:** Sử dụng thuật toán **Conflict-free Replicated Data Types (thông qua Yjs)**. Điều này cho phép nhiều người cùng chỉnh sửa một tài liệu mà không bao giờ xảy ra xung đột dữ liệu (conflict), tương tự như Google Docs nhưng hỗ trợ cả offline.

### 3. Các kỹ thuật chính (Key Techniques)
*   **OctoBase:** Một engine dữ liệu viết bằng Rust, tối ưu cho việc lưu trữ cục bộ và đồng bộ hóa thời gian thực.
*   **Turbo-renderer:** Kỹ thuật render (hiển thị) hiệu suất cao cho các canvas phức tạp, kết hợp giữa DOM và Canvas để xử lý hàng ngàn đối tượng đồ họa mượt mà.
*   **Hệ thống Adapter đa năng:** Mã nguồn cho thấy các bộ chuyển đổi mạnh mẽ (**Markdown, PDF, Notion-HTML, Plain Text**). Điều này giúp AFFiNE có khả năng nhập/xuất dữ liệu linh hoạt từ các nền tảng khác.
*   **Modular AI Partner:** Tích hợp AI trực tiếp vào quy trình làm việc (tóm tắt văn bản, vẽ mindmap từ outline, viết code) thay vì chỉ là một khung chat rời rạc.
*   **Monorepo Management:** Tổ chức mã nguồn dưới dạng monorepo (trong thư mục `packages/` và `blocksuite/`), giúp tái sử dụng mã nguồn giữa Web, Desktop và Server một cách nhất quán.

### 4. Tóm tắt luồng hoạt động (Operation Flow)
Luồng xử lý dữ liệu trong AFFiNE có thể tóm tắt như sau:

1.  **Tương tác người dùng:** Người dùng thực hiện thao tác (gõ chữ, vẽ hình) trên giao diện được xây dựng bằng **BlockSuite**.
2.  **Xử lý trạng thái (State):** Thao tác này tạo ra các bản cập nhật (updates) theo cấu trúc **Yjs**.
3.  **Lưu trữ cục bộ:** Các thay đổi ngay lập tức được ghi vào **OctoBase** (IndexedDB trên trình duyệt hoặc SQLite trên Desktop) để đảm bảo không mất dữ liệu.
4.  **Đồng bộ hóa:** Nếu có kết nối mạng và đang trong chế độ cộng tác, các bản cập nhật Yjs sẽ được gửi qua **WebSocket (hoặc WebRTC)** đến các client khác và server.
5.  **Hợp nhất dữ liệu:** Thuật toán CRDT tự động hợp nhất các thay đổi từ nhiều nguồn khác nhau mà không cần server trung tâm giải quyết xung đột.
6.  **Chuyển đổi hình thức:** Khi cần, hệ thống **Adapter** sẽ chuyển đổi cấu trúc khối hiện tại sang các định dạng khác như Markdown để chia sẻ hoặc lưu trữ bên ngoài.

**Kết luận:** AFFiNE không chỉ là một ứng dụng ghi chú, mà là một "hệ điều hành kiến thức" (Knowledge OS) mã nguồn mở, tập trung vào quyền riêng tư và hiệu năng tối đa nhờ sức mạnh của Rust và tư duy Local-first.