Dưới đây là phân tích chi tiết về dự án **Ever Teams Platform**, một nền tảng quản lý dự án và công việc mã nguồn mở dựa trên dữ liệu từ cấu trúc kho lưu trữ và tài liệu đi kèm:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng mô hình **Monorepo** với các công nghệ hiện đại nhất trong hệ sinh thái JavaScript/TypeScript:

*   **Ngôn ngữ chủ đạo:** **TypeScript (97.1%)** đảm bảo tính an toàn và nhất quán về kiểu dữ liệu trên toàn bộ hệ thống.
*   **Quản lý Monorepo:** Kết hợp **Nx**, **Lerna** và **Turbo** để tối ưu hóa quá trình build, caching và quản lý sự phụ thuộc giữa các gói.
*   **Frontend (Web):** **Next.js 16 (App Router)** và **React 19**. Sử dụng **next-intl** cho đa ngôn ngữ (i18n).
*   **Mobile:** **React Native** kết hợp với **Expo** (dựa trên boilerplate Ignite).
*   **Desktop:** **Electron** (bao bọc phiên bản web).
*   **Browser Extension:** **Plasmo Framework** để xây dựng tiện ích mở rộng cho Chrome/Edge.
*   **Styling:** **Tailwind CSS 4**, kết hợp với các thư viện component như **shadcn/ui**, **Radix UI**, và **Headless UI**.
*   **Quản lý trạng thái & Dữ liệu:** **Jotai** (Atoms) cho state global và **TanStack Query (React Query)** để quản lý việc gọi API và caching.
*   **Hệ thống Backend:** Dự án này chủ yếu đóng vai trò là Client/Interface, kết nối tới **Ever Gauzy API** (NestJS) để xử lý logic phía server và cơ sở dữ liệu.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Ever Teams được thiết kế theo hướng **Decoupled (Tách biệt hoàn toàn)** và **Reusability (Tái sử dụng cao)**:

*   **Phân tầng Ứng dụng & Gói dùng chung:**
    *   `apps/`: Chứa các ứng dụng đầu cuối (Web, Mobile, Desktop, Extension).
    *   `packages/`: Chứa logic lõi dùng chung như `services` (gọi API), `hooks` (logic React), `types` (định nghĩa dữ liệu), và `ui` (thành phần giao diện dùng chung). Điều này giúp logic tính toán thời gian (timer) hoặc quản lý tác vụ đồng nhất trên mọi nền tảng.
*   **Thiết kế hướng Feature:** Trong ứng dụng Web, mã nguồn được tổ chức theo tính năng (`core/components/features/`), giúp dễ dàng mở rộng hoặc bảo trì từng module riêng lẻ như Chat, Daily Plan, Kanban.
*   **Cơ chế "Wrapper" cho Desktop/Server-Web:** Thay vì viết lại, ứng dụng Desktop và Server-Web đóng gói ứng dụng Next.js, giúp tận dụng tối đa code frontend hiện có.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Optimistic UI (Cập nhật lạc quan):** Đặc biệt quan trọng trong tính năng Time Tracking. Khi người dùng nhấn "Start" timer, giao diện cập nhật ngay lập tức (`use-timer-optimistic-ui.ts`) trong khi yêu cầu API được xử lý bất đồng bộ phía sau.
*   **Xử lý Đa nền tảng (Cross-platform Logic):** Sử dụng các gói `packages/services` và `packages/hooks` để đảm bảo cùng một logic nghiệp vụ chạy trên cả Web và Mobile mà không phải viết lại.
*   **Xử lý Lỗi & Giám sát:** Tích hợp sâu **Sentry** trên cả Web và Mobile để theo dõi hiệu suất và lỗi runtime.
*   **Bảo mật:** Sử dụng **NextAuth.js v5** cho luồng xác thực, hỗ trợ nhiều phương thức (Magic Code, Password, Social OAuth) thông qua việc kết nối trực tiếp với backend Gauzy.
*   **Tự động hóa Docker:** Cung cấp nhiều tệp `docker-compose` cho các mục đích khác nhau (Dev có hot-reload, Demo với ảnh build sẵn, Infra cho cơ sở dữ liệu).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Xác thực (Auth):** Người dùng đăng nhập qua Web app hoặc Mobile. NextAuth gửi yêu cầu đến Gauzy API. Token JWT được lưu trữ để xác thực các yêu cầu sau đó.
2.  **Khởi tạo (Initialization):** Ứng dụng tải thông tin tổ chức (`current-org`), nhóm (`team`) và hồ sơ thành viên. Dữ liệu được cache bởi TanStack Query.
3.  **Quản lý Công việc:**
    *   Người dùng tạo Task trên bảng Kanban hoặc List view.
    *   Logic `use-task-input.ts` xử lý dữ liệu đầu vào, validate qua Zod và gửi lên server.
4.  **Theo dõi Thời gian (Time Tracking):**
    *   Khi nhấn Start, một timer global được kích hoạt.
    *   Dữ liệu thời gian được đồng bộ liên tục qua WebSockets (Socket.io) để đảm bảo nếu người dùng mở Web và Mobile đồng thời, timer sẽ hiển thị giống nhau.
5.  **Báo cáo & Năng suất:** Hệ thống thu thập dữ liệu từ các phiên làm việc (Time logs) để tạo ra các báo cáo Productivity, Timesheets và thống kê Weekly Limit.
6.  **Tích hợp (Integrations):** Khi có sự kiện từ GitHub/GitLab, backend Gauzy cập nhật và Frontend Ever Teams sẽ hiển thị các cập nhật đó trong luồng hoạt động của Task.

**Kết luận:** Ever Teams là một ví dụ điển hình về việc xây dựng hệ thống quản lý năng suất đa nền tảng hiện đại, tập trung vào việc tối ưu hóa trải nghiệm người dùng thông qua việc tái sử dụng mã nguồn và các kỹ thuật xử lý trạng thái bất đồng bộ mạnh mẽ.