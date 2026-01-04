Dựa trên mã nguồn và tài liệu của dự án **Cal-ID** (một bản fork từ Cal.com), dưới đây là phân tích chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Stack)
Dự án được xây dựng trên một hệ sinh thái JavaScript hiện đại, tập trung vào hiệu năng và tính an toàn về kiểu dữ liệu:
*   **Framework:** [Next.js](https://nextjs.org/) (sử dụng cả Pages Router và App Router) là khung sườn chính cho cả Frontend và Backend (API Routes).
*   **Giao tiếp Client-Server:** [tRPC](https://trpc.io/) cung cấp khả năng gọi API với Type-safety (an toàn kiểu dữ liệu) tuyệt đối từ server tới client mà không cần định nghĩa Schema trung gian.
*   **Cơ sở dữ liệu & ORM:** [PostgreSQL](https://www.postgresql.org/) kết hợp với [Prisma](https://prisma.io/). Prisma đóng vai trò quản lý Schema, Migrate và truy vấn dữ liệu theo hướng đối tượng.
*   **Styling:** [Tailwind CSS](https://tailwindcss.com/) kết hợp với các thư viện UI component nội bộ (nằm trong `packages/ui`).
*   **Xử lý thời gian:** [Day.js](https://day.js.org/) là thư viện chủ đạo để xử lý múi giờ và tính toán lịch trình phức tạp.
*   **Video Call:** Tích hợp sâu với [Daily.co](https://daily.co/) cho các cuộc họp video trực tuyến mặc định.
*   **Quản lý Monorepo:** Sử dụng [Turbo](https://turbo.build/) và [Yarn Workspaces](https://yarnpkg.com/features/workspaces) để quản lý hàng chục package và ứng dụng trong cùng một repository.

### 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Cal-ID rất tinh vi, phản ánh tư duy của một nền tảng quy mô lớn (Enterprise-ready):
*   **Kiến trúc Monorepo:** Tách biệt rõ ràng giữa ứng dụng (`apps/web`, `apps/api`) và logic dùng chung (`packages/`). Điều này giúp tái sử dụng mã nguồn cho các nền tảng khác nhau (Web, Mobile, Embed).
*   **Open Core:** Duy trì lõi mã nguồn mở (AGPLv3) nhưng có các module Enterprise (`/ee`) riêng biệt dành cho các tính năng thương mại (như SSO, Team Impersonation, Organizations).
*   **API-First & Headless:** Dự án không chỉ là một trang web đặt lịch, mà còn là một hạ tầng (Infrastructure). Nó hỗ trợ "Headless Router" cho phép điều hướng khách hàng dựa trên dữ liệu form mà không cần giao diện đặt lịch truyền thống.
*   **Module hóa Integrations:** Hệ thống "App Store" nội bộ (`packages/app-store`) cho phép cài đặt/gỡ bỏ các tích hợp bên thứ ba (Google Calendar, Zoom, Stripe, HubSpot) một cách độc lập thông qua giao diện plugin.

### 3. Các kỹ thuật chính (Key Techniques)
*   **PBAC (Permission-Based Access Control):** Thay vì chỉ dùng Role (Admin/User), hệ thống đang chuyển dịch sang quản lý quyền dựa trên hành động cụ thể trên tài nguyên (ví dụ: `team.create`, `booking.read`).
*   **Tính toán Availability phức tạp:** Thuật toán xử lý chồng lấn lịch (conflict checking) giữa nhiều lịch ngoại vi (Google, Outlook) và lịch nội bộ để đưa ra các khung giờ trống chính xác.
*   **Caching & Performance:** Sử dụng `calendar-cache` để giảm tải cho các API lịch bên thứ ba và tối ưu hóa tốc độ tải trang bằng Next.js SSR/SSG.
*   **Embeddable Scheduling:** Kỹ thuật đóng gói bộ đặt lịch thành các Snippet JavaScript hoặc React Component để bên thứ ba có thể nhúng vào website của họ chỉ với vài dòng code.
*   **Workflow Automation:** Hệ thống tự động hóa cho phép gửi Email/SMS nhắc hẹn dựa trên các sự kiện (Trigger) như: trước cuộc họp 24h, sau khi hủy lịch, v.v.

### 4. Tóm tắt luồng hoạt động (Hoạt động dựa trên Readme)
Dự án vận hành qua 4 giai đoạn chính:

1.  **Thiết lập (Setup):** Người dùng kết nối các lịch cá nhân (Google, Outlook), định nghĩa khung giờ làm việc (Availability) và tạo các loại sự kiện (Event Types) với các quy tắc riêng (số người tham gia, vị trí, giá tiền).
2.  **Điều phối (Routing):** Khi một khách hàng truy cập, họ có thể đi qua một "Routing Form". Dựa trên câu trả lời, hệ thống sẽ điều phối khách hàng đến đúng nhân viên hoặc loại sự kiện phù hợp (ví dụ: khách hàng VIP gặp Manager).
3.  **Đặt lịch (Booking Flow):**
    *   Khách hàng chọn khung giờ trống. Hệ thống kiểm tra thời gian thực với lịch ngoại vi để tránh trùng lặp.
    *   Hệ thống tạm giữ slot (Slot Reservation) để tránh "Race Condition" (nhiều người cùng đặt một lúc).
    *   Xác nhận đặt lịch (có thể cần thanh toán qua Stripe hoặc phê duyệt thủ công từ chủ phòng).
4.  **Xử lý sau đặt lịch (Post-Booking):** Hệ thống tự động tạo sự kiện trên lịch của các bên, tạo phòng họp video, gửi email xác nhận và kích hoạt các Workflow (nhắc hẹn qua SMS/WhatsApp).

---
**Điểm đặc biệt của Cal-ID (OneHashAI fork):** Dựa trên file cấu trúc, bản fork này bổ sung các module như `onehash-chat`, `whatsapp-business` và tích hợp sâu hơn với các hệ thống ERP/CRM của hệ sinh thái OneHash, biến nó thành một công cụ đặt lịch dành riêng cho môi trường doanh nghiệp tập trung vào bán hàng và hỗ trợ khách hàng.