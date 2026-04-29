Dựa trên phân tích cấu trúc mã nguồn và các tệp tin cấu hình của dự án **Colanode**, dưới đây là báo cáo chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng mô hình Monorepo (quản lý nhiều ứng dụng trong một kho mã nguồn) với các công nghệ hiện đại:

*   **Ngôn ngữ chủ đạo:** **TypeScript (98.7%)** được sử dụng xuyên suốt từ Backend đến Frontend, đảm bảo tính nhất quán về kiểu dữ liệu.
*   **Backend (Server):** 
    *   **Fastify:** Framework web tốc độ cao cho Node.js.
    *   **PostgreSQL + pgvector:** Cơ sở dữ liệu chính, hỗ trợ lưu trữ vector cho các tính năng AI.
    *   **Redis (Valkey):** Dùng làm Event Bus và hàng đợi công việc (BullMQ).
*   **Frontend (Web & Desktop):** 
    *   **React + Vite:** Xây dựng giao diện người dùng.
    *   **TailwindCSS + Shadcn UI:** Framework CSS và bộ thành phần giao diện.
    *   **Electron:** Đóng gói ứng dụng Desktop.
    *   **Expo (React Native):** Cho phiên bản Mobile (đang phát triển).
*   **Dữ liệu cục bộ (Client-side DB):** 
    *   **SQLite:** Sử dụng làm bộ nhớ đệm và lưu trữ chính trên thiết bị người dùng.
    *   **Kysely:** Query builder an toàn về kiểu dữ liệu cho cả Postgres và SQLite.
*   **Cơ chế cộng tác:** 
    *   **Yjs:** Thư viện CRDT (Conflict-free Replicated Data Types) cho phép nhiều người chỉnh sửa tài liệu cùng lúc mà không xung đột.
*   **Quản lý Monorepo:** **Turborepo** giúp tối ưu hóa việc build và kiểm thử.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Colanode xoay quanh triết lý **Local-first (Ưu tiên cục bộ)**:

*   **Offline-first:** Mọi dữ liệu được đọc và ghi vào SQLite cục bộ trước. Người dùng có thể làm việc ngay cả khi không có mạng.
*   **Cấu trúc Package dùng chung:** Các logic cốt lõi được tách thành các gói (`packages/`):
    *   `core`: Định nghĩa schema (Zod), quyền hạn và các kiểu dữ liệu chung.
    *   `client`: Chứa các service xử lý đồng bộ, mutation và truy vấn cho phía khách hàng.
    *   `crdt`: Lớp bao bọc (wrapper) cho Yjs để quản lý tài liệu cộng tác.
    *   `ui`: Chứa các React component dùng chung cho Web, Desktop và Mobile.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Sử dụng Redis ở phía Server và một Event Bus nội bộ ở phía Client để cập nhật giao diện thời gian thực khi có thay đổi dữ liệu.
*   **Sử dụng Cursor-based Sync:** Đồng bộ dữ liệu dựa trên con trỏ (revision cursor), giúp client chỉ tải về những thay đổi mới nhất kể từ lần đồng bộ cuối.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý xung đột bằng CRDT:** Thay vì dùng cơ chế khóa (locking), hệ thống sử dụng CRDT để tự động hợp nhất (merge) các thay đổi từ nhiều người dùng khác nhau theo thời gian thực.
*   **Optimistic Updates (Cập nhật lạc quan):** Khi người dùng thao tác, giao diện cập nhật ngay lập tức. Việc gửi dữ liệu lên server diễn ra ngầm sau đó.
*   **Phân quyền dựa trên Node:** Quyền hạn (viewer, editor, admin, owner) được tính toán dựa trên cấu trúc cây của các "Node" (Page, Database, Chat...).
*   **Resumable Uploads:** Sử dụng giao thức **TUS** để hỗ trợ tải lên tệp tin lớn, có khả năng tiếp tục nếu bị gián đoạn mạng.
*   **Vector Search:** Sử dụng `pgvector` để nhúng (embed) nội dung tài liệu vào không gian vector, phục vụ cho việc tìm kiếm thông minh và hỗ trợ trợ lý AI.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng dữ liệu của Colanode diễn ra theo chu trình khép kín giữa thiết bị người dùng và máy chủ:

1.  **Ghi dữ liệu (Write Path):** 
    *   Người dùng thực hiện thay đổi (ví dụ: gõ văn bản). 
    *   Ứng dụng áp dụng thay đổi vào SQLite cục bộ và tạo ra một bản cập nhật CRDT (binary).
    *   Bản cập nhật này được lưu vào bảng `mutations` chờ xử lý.
    *   `MutationService` sẽ gom lô (batch) và gửi lên Server qua HTTP.
2.  **Xử lý tại Server:** 
    *   Server nhận mutation, kiểm tra quyền hạn và tính hợp lệ qua Zod schema.
    *   Dữ liệu được ghi vào PostgreSQL.
    *   Server phát (broadcast) sự kiện thay đổi qua WebSocket tới các client khác đang trực tuyến.
3.  **Đọc và Đồng bộ (Read/Sync Path):**
    *   Giao diện người dùng luôn đọc dữ liệu từ SQLite (phản hồi tức thì).
    *   Một dịch vụ chạy nền (`Synchronizer`) duy trì kết nối WebSocket với Server.
    *   Khi có dữ liệu mới, Server gửi thông báo -> Client yêu cầu tải các bản ghi mới dựa trên Cursor hiện tại.
    *   Dữ liệu mới được nạp vào SQLite và giao diện tự động cập nhật qua cơ chế Reactive (TanStack Query/DB).

**Kết luận:** Colanode là một hệ thống phức tạp nhưng có kiến trúc rất chặt chẽ, kết hợp sức mạnh của cơ sở dữ liệu truyền thống với các kỹ thuật cộng tác hiện đại để tạo ra một không gian làm việc an toàn, riêng tư và hiệu suất cao.