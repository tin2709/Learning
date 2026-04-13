Dựa trên các tệp tin mã nguồn và cấu trúc thư mục của dự án **Super Productivity**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và cơ chế hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng mô hình **Cross-platform** với một lõi ứng dụng web duy nhất chạy trên nhiều nền tảng:

*   **Frontend Framework:** Angular (phiên bản mới nhất v21+), sử dụng TypeScript (chiếm >91% mã nguồn).
*   **Quản lý trạng thái (State Management):** **NgRx** (Redux pattern) là xương sống của ứng dụng, kết hợp với Angular Signals để tối ưu hóa hiệu năng UI.
*   **Nền tảng Desktop:** **Electron** để đóng gói ứng dụng cho Windows, macOS và Linux, cho phép truy cập các tính năng hệ thống như Global Shortcuts, Idle Time detection (phát hiện thời gian rảnh).
*   **Nền tảng Mobile:** **Capacitor** để đóng gói cho Android (Kotlin) và iOS (Swift). Dự án có các plugin native tự viết để xử lý WebDAV HTTP và Scoped Storage (SAF).
*   **Lưu trữ dữ liệu:** Local-first. Dữ liệu chính được lưu trong **IndexedDB** (thông qua trình duyệt/app) hoặc tệp tin cục bộ.
*   **Đồng bộ hóa (Sync):** Sử dụng hệ thống **SuperSync** (Server viết bằng Node.js + Prisma + PostgreSQL) hoặc các bên thứ ba như Dropbox, WebDAV.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Ứng dụng đi theo triết lý **"Local-first, Sync-later"** (Ưu tiên cục bộ, đồng bộ sau):

*   **Kiến trúc hướng sự kiện (Event-Driven):** Mọi tương tác của người dùng đều chuyển thành các **NgRx Actions**. Các Actions này không chỉ thay đổi UI mà còn được ghi lại vào một **Operation Log (OpLog)**.
*   **Kiến trúc Phân lớp (Layered Architecture):**
    *   *Persistence Layer:* Quản lý việc lưu trữ vào IndexedDB và xử lý OpLog.
    *   *Logic Layer (Services):** Chứa các nghiệp vụ như tính toán thời gian, phân tích cú pháp task.
    *   *Presentation Layer:* Các Component Angular sử dụng `OnPush` change detection để tối ưu hiệu năng.
*   **Mô hình Đồng bộ dựa trên Operation (OpLog-based Sync):** Thay vì gửi toàn bộ trạng thái (Full State), ứng dụng chỉ gửi các thao tác thay đổi (ví dụ: "Sửa tên task A"). Điều này giúp giảm băng thông và dễ giải quyết xung đột hơn.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Vector Clocks:** Sử dụng kỹ thuật Vector Clocks trong tệp `vector-clock.ts` để theo dõi thứ tự các thao tác và giải quyết xung đột khi đồng bộ hóa giữa nhiều thiết bị mà không cần server trung tâm điều phối thời gian.
*   **Meta-Reducers cho tính nguyên tử (Atomicity):** Sử dụng Meta-reducers (`task-shared-meta-reducers`) để đảm bảo khi một hành động xảy ra (ví dụ: xóa một Tag), tất cả các thực thể liên quan (các Task đang mang Tag đó) đều được cập nhật trong cùng một chu kỳ xử lý, tránh mất dữ liệu.
*   **Validation thời gian thực:** Sử dụng thư viện **Typia** để kiểm tra kiểu dữ liệu (runtime validation) cho các đối tượng JSON phức tạp, đảm bảo dữ liệu từ các nguồn đồng bộ bên ngoài không làm hỏng trạng thái ứng dụng.
*   **Virtual Tags (Thẻ ảo):** Một kỹ thuật thông minh được ghi trong `ARCHITECTURE-DECISIONS.md`. Thẻ "TODAY" không thực sự tồn tại trong mảng `tagIds` của Task, mà được tính toán động dựa trên ngày đến hạn (`dueDay`), giúp việc quản lý danh sách việc cần làm trong ngày luôn nhất quán.
*   **Native Bridge:** Sử dụng `@JavascriptInterface` (Android) để tạo cầu nối giữa JavaScript và mã Kotlin, cho phép ứng dụng web điều khiển các Foreground Service của Android để theo dõi thời gian chính xác ngay cả khi app chạy ngầm.

### 4. Luồng hoạt động của hệ thống (System Workflow)

#### A. Luồng xử lý Task và Thời gian:
1.  Người dùng nhập task mới -> **Action `addTask`** được dispatch.
2.  **Meta-reducer** bắt lấy Action -> Ghi thông tin vào **IndexedDB** + Ghi một bản ghi vào **OpLog**.
3.  **Selector** cập nhật UI ngay lập tức.
4.  Nếu bật tính năng theo dõi thời gian -> Một **Foreground Service** (trên Mobile) hoặc **Electron Timer** (trên Desktop) được kích hoạt để đếm giờ độc lập với luồng UI.

#### B. Luồng đồng bộ hóa (SuperSync):
1.  **Capture:** Mỗi khi trạng thái thay đổi, hệ thống lưu lại thao tác vào OpLog cục bộ.
2.  **Upload:** Khi có mạng, ứng dụng đẩy các Ops mới nhất lên SuperSync Server.
3.  **Download & Replay:** Ứng dụng tải các Ops từ thiết bị khác về, sử dụng **Vector Clocks** để kiểm tra xung đột.
4.  **Conflict Resolution:** Nếu hai thiết bị cùng sửa một task, quy tắc "Last Write Wins" hoặc dựa trên Vector Clock sẽ quyết định bản sửa nào được giữ lại.
5.  **Hydration:** Sau khi áp dụng các thay đổi từ xa, ứng dụng cập nhật lại toàn bộ Store để UI phản ánh đúng dữ liệu mới.

### Tóm kết
**Super Productivity** là một dự án có độ phức tạp kỹ thuật rất cao, đặc biệt là trong việc giải quyết bài toán đồng bộ dữ liệu phân tán (Distributed Data Sync) trên nền tảng Web. Việc kết hợp giữa sự linh hoạt của Angular và sức mạnh hệ thống của Electron/Capacitor tạo ra một công cụ làm việc mạnh mẽ, quyền riêng tư cao (Local-first).