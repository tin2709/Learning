Dựa trên phân tích mã nguồn và tài liệu kiến trúc của **InstantDB**, đây là bản tóm tắt chuyên sâu về hệ thống được mệnh danh là "Firebase hiện đại" này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Backend (Sync Server):** Sử dụng **Clojure** (~43%) làm ngôn ngữ chính. Clojure cực kỳ mạnh mẽ trong việc xử lý dữ liệu bất biến (immutable) và concurrency, phù hợp cho việc duy trì hàng ngàn kết nối WebSocket thời gian thực.
*   **Cơ sở dữ liệu:** **PostgreSQL** là trung tâm. Tuy nhiên, InstantDB không dùng Postgres theo cách truyền thống mà lưu trữ dữ liệu dưới dạng **Triples (Entity-Attribute-Value)**. Điều này cho phép mở rộng schema linh hoạt mà không cần chạy `ALTER TABLE`.
*   **Frontend SDK:** Viết bằng **TypeScript** (~51%), hỗ trợ đa nền tảng từ Web (React, Next.js, Svelte, SolidJS) đến Mobile (React Native).
*   **Hệ thống quyền hạn:** Sử dụng thư viện **Google CEL (Common Expression Language)** trên JVM (Java) để định nghĩa và thực thi các quy tắc bảo mật nhanh chóng và an toàn.
*   **Lưu trữ Client:** Sử dụng **IndexedDB** (trên Web) và **AsyncStorage/MMKV** (trên Mobile) để hỗ trợ chế độ Offline-first và lưu trữ cache.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của InstantDB tập trung vào việc **"Nén các công đoạn phiền hà" (Compressing Schleps)**:

*   **Graph-Database-on-the-Client:** Thay vì gọi API REST/GraphQL, lập trình viên truy vấn dữ liệu theo dạng đồ thị (relational queries) ngay tại client. Client SDK hoạt động như một cơ sở dữ liệu đồ thị nhỏ đồng bộ với server.
*   **Multi-tenant Triplestore:** Một cơ sở dữ liệu Postgres lớn chứa dữ liệu của nhiều ứng dụng khác nhau. Mỗi mẩu dữ liệu là một "triple": `[Entity ID, Attribute, Value]`. Cách tiếp cận này giúp Instant cung cấp tính năng "schema-less" nhưng vẫn đảm bảo tính toàn vẹn của dữ liệu quan hệ.
*   **Reactive Loop:** Hệ thống được thiết kế để mọi truy vấn đều có tính năng "Live". Khi dữ liệu thay đổi ở Postgres, một luồng phản ứng sẽ đẩy dữ liệu mới đến tất cả các client đang quan tâm đến truy vấn đó.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **WAL Tailing (Write-Ahead Log):** Server của InstantDB "theo đuôi" (tail) file log ghi trước của PostgreSQL. Khi có bất kỳ thay đổi nào trong DB, server phát hiện ngay lập tức để tính toán các truy vấn cần bị hủy bỏ (invalidation) và cập nhật cho client.
*   **Optimistic Updates & Rollbacks:** Khi client thực hiện `transact`, SDK cập nhật ngay lập tức vào bộ nhớ cục bộ (optimistic) để tạo cảm giác cực nhanh. Nếu server từ chối giao dịch (do quyền hạn hoặc lỗi mạng), SDK tự động thực hiện rollback trạng thái về điểm nhất quán cuối cùng.
*   **Functional Effect System (CLI):** Công cụ CLI (`instant-cli`) được xây dựng trên thư viện **Effect-TS**, giúp quản lý lỗi và xử lý bất đồng bộ theo phong cách lập trình hàm (Functional Programming) chặt chẽ.
*   **Datalog-inspired Query Engine:** Ngôn ngữ **InstaQL** được lấy cảm hứng từ Datalog, cho phép lấy các cấu trúc dữ liệu lồng nhau phức tạp chỉ trong một lần gọi mà không gặp vấn đề N+1 query.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Ghi (Write Workflow - InstaML):
1.  **Client:** Gọi hàm `db.transact([...])`.
2.  **SDK:** Ghi thay đổi vào cache local ngay lập tức và gửi gói tin giao dịch lên server qua WebSocket/HTTP.
3.  **Server:** 
    *   Xác thực người dùng.
    *   Chạy quy tắc **CEL Permissions** để kiểm tra quyền ghi.
    *   Nếu đạt, chuyển đổi lệnh ghi thành các bản ghi Triples và lưu vào Postgres.
4.  **Confirm:** Server phản hồi thành công. Nếu lỗi, SDK tự xóa bỏ thay đổi tạm thời ở bước 2.

#### B. Luồng Đọc & Đồng bộ (Read & Sync Workflow - InstaQL):
1.  **Client:** Sử dụng hook `useQuery` để đăng ký một truy vấn.
2.  **SDK:** Kiểm tra dữ liệu trong IndexedDB/Cache. Gửi yêu cầu "Subscribe" lên Sync Server.
3.  **Server:** Thực hiện truy vấn Datalog xuống Postgres, trả về kết quả ban đầu và duy trì một subscription trong bộ nhớ.
4.  **Novelty Detection:** Khi một người dùng khác ghi dữ liệu, Sync Server đọc WAL log từ Postgres.
5.  **Re-computation:** Server xác định xem thay đổi đó có ảnh hưởng đến các client đang subcribe hay không và chỉ gửi phần dữ liệu thay đổi (diff) xuống.

### Tổng kết
InstantDB là một nỗ lực đưa sức mạnh của **Clojure và Graph Database** vào tay các lập trình viên Web/Mobile thông qua lớp vỏ bọc **TypeScript** đơn giản. Kỹ thuật then chốt là việc sử dụng **Postgres WAL** để biến một database truyền thống thành một engine thời gian thực mạnh mẽ.