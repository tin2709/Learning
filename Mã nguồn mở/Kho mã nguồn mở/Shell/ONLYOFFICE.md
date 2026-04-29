Dựa trên các tài liệu và cấu trúc kho lưu trữ (repository) của **ONLYOFFICE DocumentServer** (hiện gọi là ONLYOFFICE Docs), dưới đây là phân tích chi tiết:

### 1. Công nghệ cốt lõi (Core Technologies)

ONLYOFFICE Docs là một hệ thống phức hợp đa ngôn ngữ, kết hợp giữa hiệu năng tính toán thấp và trải nghiệm web hiện đại:

*   **Backend (Server Layer):** Sử dụng **Node.js** làm lớp điều phối (orchestration), quản lý kết nối và các tác vụ thời gian thực.
*   **Core (Computational Layer):** Được viết bằng **C++**. Đây là "trái tim" của hệ thống, chịu trách nhiệm chuyển đổi (conversion) giữa các định dạng (DOCX, XLSX, PPTX, PDF...) và xử lý logic bố cục văn bản cực kỳ phức tạp.
*   **Frontend & Client SDK:** 
    *   **JavaScript/TypeScript:** Sử dụng **SDKJS** để tương tác với nhân xử lý.
    *   **ReactJS:** Được áp dụng mạnh mẽ cho phiên bản Mobile Web (theo Roadmap).
*   **Định dạng chuẩn:** Tập trung tuyệt đối vào **OOXML** (.docx, .xlsx, .pptx) làm định dạng gốc, đảm bảo độ tương thích cao nhất với Microsoft Office.
*   **Cơ sở hạ tầng:** Sử dụng **RabbitMQ** (hàng đợi tin nhắn), **Redis** (caching), và **PostgreSQL** (lưu trữ dữ liệu) để vận hành quy mô lớn.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ONLYOFFICE được thiết kế theo hướng **Thanh tách biệt và Module hóa cao (Decoupled & Modular)**:

*   **Kiến trúc Client-side Rendering:** Khác với nhiều bộ office online khác render ảnh ở server rồi gửi về, ONLYOFFICE đẩy phần lớn việc xử lý hiển thị về phía trình duyệt của khách hàng. Trình duyệt tải toàn bộ engine xử lý (SDKJS) để tự render văn bản.
*   **Cấu trúc 5 thành phần chính:**
    1.  `server`: Lớp backend Node.js.
    2.  `core`: Nhân xử lý định dạng C++.
    3.  `sdkjs`: Thư viện API cho phía client.
    4.  `web-apps`: Giao diện người dùng (UI) và trình biên tập.
    5.  `dictionaries`: Hệ thống từ điển đa ngôn ngữ.
*   **Kiến trúc Monorepo (quản lý qua Submodules):** Tệp `.mrconfig` và `mr-update.sh` cho thấy dự án sử dụng công cụ `myrepos` để quản lý hàng chục kho lưu trữ con khác nhau, cho phép phát triển độc lập từng thành phần nhưng vẫn đồng bộ hóa được toàn bộ hệ thống.
*   **Khả năng nhúng (Integrator-first):** Thiết kế cho phép dễ dàng tích hợp vào các nền tảng bên thứ ba (Nextcloud, Moodle, Odoo...) thông qua các Connectors.

### 3. Các kỹ thuật chính (Key Techniques)

*   **HTML5 Canvas Rendering:** Đây là kỹ thuật đặc trưng nhất. ONLYOFFICE sử dụng Canvas để vẽ nội dung văn bản. Điều này đảm bảo tài liệu hiển thị **giống hệt nhau 100% (WYSIWYG)** trên mọi trình duyệt và hệ điều hành, tránh lỗi vỡ định dạng do font hệ thống.
*   **Real-time Collaborative Editing (Đồng soạn thảo thời gian thực):**
    *   **Fast mode:** Hiển thị thay đổi ngay lập tức (như Google Docs).
    *   **Strict mode:** Chỉ hiển thị thay đổi sau khi người dùng nhấn "Save" (giúp tập trung hơn, tránh bị nhiễu bởi người khác).
*   **Chuyển đổi định dạng trung gian:** Khi một file (ví dụ .odt) được mở, `core` sẽ chuyển nó sang một cấu trúc JSON nội bộ để client xử lý, sau đó khi lưu sẽ đóng gói lại định dạng mục tiêu.
*   **Security & Encryption:** Hỗ trợ **Private Rooms** (E2EE - mã hóa đầu cuối) và tích hợp JWT để xác thực các yêu cầu giữa DMS (Document Management System) và Document Server.
*   **Hệ thống Plugin:** Cho phép mở rộng tính năng (như AI Assistant, Zotero) mà không làm phình to bộ mã nguồn lõi.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Yêu cầu mở file:** Người dùng từ một nền tảng (ví dụ Nextcloud) click mở file. Nền tảng này gửi một `config` (chứa URL file và token) tới Document Server.
2.  **Khởi tạo Client:** Document Server gửi `web-apps` (JS, CSS) và `SDKJS` về trình duyệt. Trình duyệt khởi tạo một khung soạn thảo trống.
3.  **Tải và Chuyển đổi:** `server` Node.js yêu cầu `core` lấy file từ nguồn, chuyển đổi nó sang định dạng dữ liệu nội bộ (JSON) và gửi về trình duyệt.
4.  **Soạn thảo:** Người dùng sửa đổi văn bản -> Trình duyệt ghi lại các "action" -> Gửi action về server qua **WebSocket**.
5.  **Đồng bộ:** Server nhận action, gửi tới các client khác đang mở cùng file đó để cập nhật giao diện Canvas của họ.
6.  **Lưu trữ:** Khi mọi người rời khỏi phiên làm việc, `server` yêu cầu `core` lắp ghép các action lại thành file hoàn chỉnh và "push" file đã lưu ngược lại nền tảng lưu trữ ban đầu (DMS).

**Kết luận:** ONLYOFFICE DocumentServer là một hệ thống cực kỳ mạnh mẽ, ưu tiên độ chính xác hiển thị (qua Canvas) và tính module hóa. Roadmap cho thấy họ đang chuyển dịch mạnh sang tích hợp AI và tối ưu hóa cho nền tảng Mobile bằng ReactJS.