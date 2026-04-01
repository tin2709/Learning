Dưới đây là phân tích chi tiết về dự án **OpenStock** dựa trên cấu trúc thư mục và mã nguồn bạn đã cung cấp:

### 1. Công nghệ cốt lõi (Core Technology Stack)

OpenStock sử dụng một Stack hiện đại, tập trung vào hiệu suất và khả năng mở rộng nhanh (Scalability):

*   **Framework chính:** **Next.js 15 (App Router)** kết hợp với **React 19**. Đây là lựa chọn hàng đầu cho các ứng dụng web hiện nay, hỗ trợ Server Components giúp tối ưu SEO và tốc độ tải trang.
*   **Ngôn ngữ:** **TypeScript (chiếm >91%)**. Việc sử dụng strict type giúp giảm thiểu lỗi runtime trong các tính toán tài chính và xử lý dữ liệu chứng khoán.
*   **Giao diện (UI/UX):** **Tailwind CSS v4**, **shadcn/ui** và **Radix UI**. Tư duy thiết kế "Utility-first" kết hợp với các thành phần không trạng thái (headless components) giúp giao diện đồng nhất và dễ tùy biến.
*   **Cơ sở dữ liệu:** **MongoDB** phối hợp với **Mongoose**. Cấu trúc NoSQL linh hoạt cho việc lưu trữ Watchlist, Alert và hồ sơ người dùng đa dạng.
*   **Xác thực (Authentication):** **Better Auth**. Một thư viện mới nổi giúp quản lý Session và User Identity an toàn, tích hợp sẵn Adapter cho MongoDB.
*   **Xử lý tác vụ nền (Background Jobs):** **Inngest**. Đây là "xương sống" cho các luồng Event-driven của hệ thống, thay thế cho các hàng đợi (queues) truyền thống phức tạp.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế theo tư duy **Resilient & Event-Driven (Bền bỉ và Hướng sự kiện)**:

*   **Chiến lược AI Đa tầng (Multi-provider AI):** Kiến trúc có khả năng tự động chuyển đổi (failover). Nếu Google Gemini (Primary) gặp lỗi hoặc giới hạn tốc độ (Rate limit), hệ thống sẽ ngay lập tức chuyển sang Siray.ai hoặc MiniMax (Fallback). Điều này đảm bảo tính năng "AI Insights" luôn khả dụng.
*   **Tư duy Serverless & Edge:** Hệ thống tận dụng tối đa Server Actions (`lib/actions`) để xử lý logic backend ngay trong Next.js, giúp giảm bớt gánh nặng quản lý server truyền thống.
*   **Kiến trúc Hướng sự kiện (EDA):** Thông qua Inngest, các hành động như đăng ký tài khoản (`app/user.created`) sẽ kích hoạt một chuỗi workflow: tạo nội dung cá nhân hóa bằng AI -> gửi email -> cập nhật tag trong marketing tool (Kit).
*   **Tách biệt dữ liệu và hiển thị:** Dữ liệu thị trường thời gian thực không được lưu vào DB cá nhân mà được "nhúng" trực tiếp từ các Widget của **TradingView** và gọi qua **Finnhub API**, giúp giảm chi phí lưu trữ và độ trễ dữ liệu.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Provider Abstraction Pattern:** Trong `lib/ai-provider.ts`, mã nguồn định nghĩa một interface chung cho các AI model khác nhau. Kỹ thuật này giúp lập trình viên thêm mới một nhà cung cấp AI chỉ bằng cách cập nhật config mà không cần sửa logic nghiệp vụ.
*   **Optimistic Updates:** Khi người dùng thêm/xóa cổ phiếu khỏi Watchlist (`WatchlistButton.tsx`), UI sẽ cập nhật ngay lập tức trước khi nhận được phản hồi từ server để tạo cảm giác mượt mà (perceived performance).
*   **Custom Hooks cho thư viện bên thứ ba:** `useTradingViewWidget.tsx` đóng gói logic tiêm (inject) script và cấu hình phức tạp của TradingView thành một React Hook dễ tái sử dụng.
*   **Debouncing:** Sử dụng `useDebounce.ts` trong tính năng tìm kiếm để giảm số lượng request tới Finnhub API, tránh lãng phí tài nguyên và dính giới hạn API.
*   **Middleware Protection:** Sử dụng Next.js Middleware (`middleware/index.ts`) để tập trung hóa logic kiểm tra quyền truy cập (Authentication/Authorization) cho toàn bộ ứng dụng.

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống vận hành theo 3 luồng chính:

**Luồng A: Cá nhân hóa người dùng (Onboarding Workflow)**
1. Người dùng đăng ký và điền khảo sát (mục tiêu đầu tư, mức độ rủi ro).
2. Hệ thống lưu dữ liệu vào MongoDB và phát ra sự kiện `app/user.created`.
3. Inngest bắt sự kiện, gọi Gemini AI để tạo lời chào cá nhân hóa dựa trên dữ liệu khảo sát.
4. Gửi Email thông qua Nodemailer/Kit.

**Luồng B: Theo dõi thị trường (Market Alert Workflow)**
1. Người dùng thiết lập giá mục tiêu (Target Price) cho một mã cổ phiếu.
2. Inngest chạy một Cron job định kỳ (mỗi 5 phút).
3. Job lấy giá hiện tại từ Finnhub API, so sánh với Target Price trong DB.
4. Nếu điều kiện khớp (Price > Target hoặc Price < Target), hệ thống gửi thông báo/email cho người dùng.

**Luồng C: Hiển thị dữ liệu thời gian thực (Real-time Dashboard)**
1. Người dùng truy cập trang Dashboard hoặc chi tiết cổ phiếu.
2. Server Actions lấy thông tin cơ bản từ MongoDB.
3. Client-side tải các script của TradingView để vẽ biểu đồ và heatmap trực tiếp từ CDN của họ.
4. Các dữ liệu "Sentiment" (cảm xúc thị trường) được fetch song song từ Adanos API để bổ sung thông tin phân tích.

### Tổng kết
OpenStock là một dự án **Modern Open Source** điển hình. Nó không cố gắng tự xây dựng lại bánh xe (biểu đồ, dữ liệu sàn) mà tập trung vào việc **điều phối (Orchestration)** các dịch vụ bên thứ ba và sử dụng **AI/Workflow automation** để tạo ra giá trị gia tăng cho người dùng cuối.