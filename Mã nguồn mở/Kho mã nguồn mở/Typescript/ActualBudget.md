Dựa trên nội dung kho lưu trữ (repository) của **Actual Budget**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào hiệu suất và khả năng chạy trên nhiều nền tảng:
*   **Ngôn ngữ lập trình:** **TypeScript (chiếm ~90%)** là chủ đạo, đảm bảo tính chặt chẽ về kiểu dữ liệu cho một hệ thống tài chính.
*   **Frontend:** **React** kết hợp với **Emotion** (CSS-in-JS) để xây dựng giao diện người dùng (UI). Sử dụng **Vite** làm công cụ đóng gói (bundler) thay vì Webpack để tăng tốc độ phát triển.
*   **Backend & Core:** **Node.js** đóng vai trò là môi trường chạy các logic nghiệp vụ nặng.
*   **Cơ sở dữ liệu:** **SQLite** (thông qua `better-sqlite3`) được sử dụng để lưu trữ dữ liệu cục bộ trên thiết bị người dùng.
*   **Desktop App:** **Electron** được dùng để đóng gói ứng dụng chạy trên Windows, macOS và Linux.
*   **Quản lý Monorepo:** Sử dụng **Yarn 4 (Workspaces)** kết hợp với **Lage** (một task runner hiệu năng cao từ Microsoft) để quản lý nhiều gói (packages) trong cùng một repo.

### 2. Tư duy kiến trúc và Kĩ thuật đặc trưng (Architectural Thinking)
Actual Budget áp dụng các tư duy kiến trúc phần mềm tiên tiến:
*   **Local-first (Ưu tiên cục bộ):** Đây là triết lý quan trọng nhất. Dữ liệu được lưu và xử lý trực tiếp trên thiết bị của người dùng. Việc đồng bộ hóa là một lớp bổ sung chứ không phải điều kiện bắt buộc để ứng dụng hoạt động. Điều này giúp ứng dụng cực nhanh và bảo mật quyền riêng tư.
*   **Tách biệt logic nghiệp vụ (Headless Core):** Gói `loot-core` chứa toàn bộ logic tính toán, quản lý ngân sách và tương tác DB. Nó được thiết kế để chạy độc lập với giao diện, có thể chạy trong trình duyệt (qua Web Workers), trong Node.js hoặc Electron.
*   **Kiến trúc Monorepo:** Chia nhỏ ứng dụng thành các module chuyên biệt:
    *   `api`: Thư viện cho lập trình viên tương tác với ứng dụng.
    *   `crdt`: Thuật toán xử lý xung đột dữ liệu.
    *   `desktop-client`: Giao diện React.
    *   `sync-server`: Server trung gian để đồng bộ hóa dữ liệu mã hóa giữa các thiết bị.
*   **Mã hóa đầu cuối (End-to-End Encryption):** Dữ liệu được mã hóa trước khi gửi lên server đồng bộ, đảm bảo ngay cả chủ sở hữu server cũng không đọc được dữ liệu tài chính của bạn.

### 3. Các kỹ thuật chính (Key Technical Features)
*   **CRDT (Conflict-free Replicated Data Types):** Sử dụng cấu trúc dữ liệu đặc biệt để giải quyết các xung đột khi người dùng nhập liệu trên nhiều thiết bị khác nhau (ví dụ: điện thoại và máy tính cùng lúc) mà không cần một server trung tâm điều phối "ai đúng ai sai".
*   **AQL (Actual Query Language):** Một lớp truy vấn tùy chỉnh (`packages/api/app/query.js`) giúp việc lọc và lấy dữ liệu tài chính trở nên trực quan và mạnh mẽ hơn so với SQL thuần túy.
*   **Protocol Buffers (Protobuf):** Sử dụng để định nghĩa cấu trúc dữ liệu đồng bộ, giúp việc truyền tải dữ liệu qua mạng cực kỳ nhỏ gọn và nhanh chóng.
*   **Lage Task Runner:** Tối ưu hóa việc build và test bằng cách lưu bộ nhớ đệm (cache) các gói đã build rồi, chỉ build lại những gì thay đổi.
*   **Hệ thống Icon tự động:** Các icon được quản lý trong `component-library` và được chuyển đổi tự động từ SVG sang React Components thông qua SVGR.

### 4. Tóm tắt luồng hoạt động (Operational Flow)
Dựa trên mô tả và cấu trúc file, luồng hoạt động của Actual Budget diễn ra như sau:

1.  **Khởi tạo:** Khi người dùng mở ứng dụng, `desktop-client` (UI) khởi động và kết nối với `loot-core` (Backend chạy ngầm).
2.  **Tương tác dữ liệu:**
    *   Người dùng nhập một giao dịch chi tiêu trên UI.
    *   UI gửi yêu cầu thông qua lớp API đến `loot-core`.
    *   `loot-core` thực hiện các phép tính toán ngân sách (theo phương pháp Envelope Budgeting - chia tiền vào các phong bì) và ghi dữ liệu vào SQLite cục bộ.
3.  **Đồng bộ hóa:**
    *   Mỗi khi có thay đổi, gói `crdt` sẽ tạo ra một "thông điệp thay đổi" có kèm dấu thời gian (timestamp).
    *   Thông điệp này được mã hóa và gửi đến `sync-server`.
    *   Khi người dùng mở ứng dụng trên thiết bị thứ hai, thiết bị này sẽ tải các thông điệp từ `sync-server`, giải mã và "hợp nhất" (merge) vào cơ sở dữ liệu cục bộ của nó.
4.  **Mở rộng (API):** Người dùng nâng cao có thể sử dụng gói `@actual-app/api` để viết script tự động hóa việc nhập giao dịch từ ngân hàng hoặc tạo báo cáo tùy chỉnh mà không cần thông qua giao diện đồ họa.

**Kết luận:** Actual Budget là một dự án có kỹ thuật rất cao, tập trung vào việc trao quyền kiểm soát dữ liệu hoàn toàn cho người dùng thông qua kiến trúc Local-first và thuật toán đồng bộ hóa CRDT phức tạp.