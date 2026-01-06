Dưới đây là bản phân tích chi tiết về dự án **Tiledesk Design Studio** dựa trên mã nguồn và tài liệu bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án được xây dựng trên nền tảng Web hiện đại, tập trung vào tính tương tác cao:

*   **Framework:** **Angular 14.2.6** (Sử dụng Ivy compiler). Đây là phiên bản ổn định, hỗ trợ tốt cho các ứng dụng doanh nghiệp lớn (Enterprise).
*   **Ngôn ngữ:** **TypeScript (56.9%)** đóng vai trò chủ đạo, đảm bảo tính chặt chẽ về dữ liệu (type-safe) cho các logic chatbot phức tạp.
*   **Giao diện & Style:** 
    *   **SCSS (20.9%)**: Sử dụng kiến trúc module hóa style (phân tách rõ rệt giữa dashboard, canvas, và các action).
    *   **Angular Material & Bootstrap 5**: Kết hợp để tận dụng các component UI chuẩn (Material) và hệ thống grid/layout linh hoạt (Bootstrap).
*   **Xử lý thời gian thực:** **MQTT & WebSockets**. Dự án tích hợp `chat21-core` để xử lý tin nhắn và trạng thái hiện diện (presence) theo thời gian thực.
*   **Lưu trữ & Auth:** Tích hợp sâu với **Firebase** (Auth, Storage) và hệ thống Token nội bộ của Tiledesk.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Engineering)

Kiến trúc của Design Studio được thiết kế theo hướng **Modular & Service-Oriented**:

*   **Kiến trúc Đồ thị (Graph-based UI):** Dự án không chỉ là các form nhập liệu mà là một "Canvas". Tư duy kiến trúc ở đây là quản lý các "Blocks" (Intents) và "Connectors" (liên kết giữa các block).
*   **Quản lý trạng thái (State Management):** Sử dụng các Angular Services (`DashboardService`, `IntentService`, `StageService`) kết hợp với **RxJS (BehaviorSubjects)** để đồng bộ dữ liệu giữa các panel điều khiển và vùng làm việc (Canvas).
*   **Tính linh hoạt của Cấu hình (Runtime Configuration):** Sử dụng cơ chế `APP_INITIALIZER` để tải file cấu hình (`design-studio-config.json`) trước khi ứng dụng khởi chạy. Điều này cho phép deploy cùng một bản build lên nhiều môi trường (Staging, Prod) chỉ bằng cách thay đổi biến môi trường.
*   **Interceptors & Guards:** 
    *   `AuthGuard` và `RoleGuard` kiểm soát quyền truy cập dựa trên JWT và vai trò (Owner, Admin).
    *   `HeadersInterceptor` tự động đính kèm token vào mọi request API.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

1.  **Hệ thống No-code Actions:** 
    *   Dự án định nghĩa một danh sách dài các `TYPE_ACTION` (Reply, Web Request, GPT Task, v.v.) trong `utils-actions.ts`. 
    *   Mỗi action là một component riêng biệt nhưng tuân thủ interface chung, giúp dễ dàng mở rộng thêm các tính năng mới mà không phá vỡ cấu trúc cũ.
2.  **Tích hợp AI/LLM linh hoạt:** 
    *   Hệ thống hỗ trợ đa nền tảng AI (OpenAI, Anthropic, Google Gemini, Groq, DeepSeek) thông qua bộ lọc trong `utils-ai_models.ts`. 
    *   Có cơ chế quản lý "Token Multiplier" để tính toán chi phí/giới hạn cho từng model AI khác nhau.
3.  **Xử lý biến (Variable Logic):** 
    *   `TiledeskVarSplitter.ts`: Kỹ thuật Regex để tách và xử lý các biến trong chuỗi văn bản (ví dụ: `${user_name}`). 
    *   Hệ thống phân loại biến thông minh: Biến hệ thống, biến người dùng định nghĩa, biến từ thuộc tính Lead.
4.  **Cá nhân hóa Thương hiệu (White-labeling):** 
    *   `BrandResources.ts`: Cho phép thay đổi toàn bộ Logo, màu sắc, tiêu đề trang dựa trên cấu hình từ Server, hỗ trợ các đối tác muốn đóng gói lại sản phẩm (Rebranding).
5.  **Công cụ lập trình trực quan (Visual Tools):** 
    *   Xử lý logic kết nối (Connectors) phức tạp: Tính toán điểm nối `True/False` của một action để vẽ đường dẫn tới Intent tiếp theo.

---

### 4. Tóm tắt luồng hoạt động của Project (Activity Flow)

Dựa trên file `README.md` và mã nguồn, luồng hoạt động chính như sau:

1.  **Khởi tạo (Initialization):** 
    *   Ứng dụng kiểm tra `tiledesk_token` trong Local Storage. 
    *   Nếu hợp lệ, `AppConfigService` tải các tham số API và `BrandService` tải giao diện thương hiệu.
2.  **Môi trường làm việc (Dashboard):** 
    *   Người dùng truy cập theo URL: `/project/<id>/chatbot/<id>/blocks`. 
    *   Hệ thống tải danh sách các Intent (ý định) và các Rules (quy tắc) của chatbot đó từ Tiledesk Backend.
3.  **Thiết kế hội thoại (Design Phase):**
    *   **Kéo-thả (Drag & Drop):** Người dùng tạo các Block. Trong mỗi Block, họ thêm các "Action" (ví dụ: gửi tin nhắn, hỏi AI, gửi Email).
    *   **Kết nối (Connecting):** Người dùng kéo đường dây nối từ Action này sang Block khác để tạo logic rẽ nhánh (ví dụ: Nếu khách hỏi về "Giá" -> Nối sang Block "Bảng giá").
4.  **Tích hợp AI (AI Powering):** 
    *   Người dùng cấu hình "Knowledge Base" (Cơ sở kiến thức) để chatbot tự học từ website/tài liệu. 
    *   Sử dụng "GPT Task" để xử lý các logic ngôn ngữ tự nhiên ngay trong luồng hội thoại.
5.  **Xuất bản (Publishing):** 
    *   Sau khi thiết kế xong, dữ liệu JSON của đồ thị hội thoại được lưu lại. Chatbot của Tiledesk sẽ đọc cấu hình này để phản hồi khách hàng trên các kênh (WebWidget, WhatsApp, Telegram).

**Kết luận:** Đây là một dự án Angular có độ phức tạp cao, kết hợp giữa tư duy thiết kế hệ thống (System Design) và trải nghiệm người dùng trực quan (UX Design), định hướng thay thế cho các nền tảng như Voiceflow hay Botpress.