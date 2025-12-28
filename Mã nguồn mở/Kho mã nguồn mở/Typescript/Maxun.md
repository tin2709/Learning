Dựa trên nội dung các tệp tin và mã nguồn bạn đã cung cấp, dưới đây là phân tích chi tiết về dự án **Maxun** theo các tiêu chí yêu cầu:

### 1. Công nghệ cốt lõi (Core Technologies)
Maxun được xây dựng trên một ngăn xếp công nghệ (stack) hiện đại, tập trung vào hiệu suất và khả năng mở rộng:

*   **Ngôn ngữ chủ đạo:** **TypeScript** (chiếm 93%) được sử dụng xuyên suốt từ Frontend đến Backend, đảm bảo tính chặt chẽ của dữ liệu.
*   **Tự động hóa trình duyệt (Browser Automation):** **Playwright** là "trái tim" của hệ thống, điều khiển các trình duyệt không đầu (headless browsers) để tương tác và trích xuất dữ liệu.
*   **Backend:** **Node.js** kết hợp với framework **Express**. Sử dụng **Sequelize** (ORM) để giao tiếp với cơ sở dữ liệu.
*   **Frontend:** **React** với công cụ build **Vite**. Thư viện giao diện là **Material UI (MUI)** và **Styled-components**.
*   **Cơ sở dữ liệu & Lưu trữ:**
    *   **PostgreSQL:** Lưu trữ dữ liệu người dùng, cấu hình robot và lịch sử chạy (runs).
    *   **Redis:** Quản lý hàng đợi tác vụ và trạng thái thời gian thực.
    *   **MinIO (S3 compatible):** Lưu trữ các tệp tin nhị phân như ảnh chụp màn hình (screenshots) từ kết quả scrape.
*   **Quản lý hàng đợi (Task Queue):** **PgBoss** được sử dụng để xử lý các tác vụ bất đồng bộ nặng như chạy robot theo lịch hoặc xử lý hàng nghìn dòng dữ liệu.
*   **Trí tuệ nhân tạo (AI):** Tích hợp SDK của **Anthropic (Claude)**, **OpenAI (GPT)** và hỗ trợ **Ollama** (AI cục bộ) để trích xuất dữ liệu bằng ngôn ngữ tự nhiên.
*   **Giao tiếp thời gian thực:** **Socket.io** truyền tải snapshot trình duyệt và nhật ký (logs) từ server về client ngay lập tức.

### 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Maxun được thiết kế theo hướng **Decoupled (Tách rời)** và **Stateful Automation**:

*   **Phân tách Core logic:** Logic trích xuất chính nằm ở gói `maxun-core`, tách biệt hoàn toàn với server và giao diện. Điều này cho phép SDK hoặc các ứng dụng khác sử dụng lại lõi này.
*   **Browser-as-a-Service:** Thay vì chạy browser trực tiếp trên web server, Maxun có một service `browser/` riêng biệt, kết nối qua WebSocket (CDP - Chrome DevTools Protocol). Tư duy này giúp cô lập tài nguyên và dễ dàng scale-out trình duyệt.
*   **Kiến trúc dựa trên Workflow (Workflow-driven):** Mọi hành động của người dùng (click, nhập liệu, cuộn) được chuyển đổi thành một danh sách các cặp "Where" (Điều kiện/Selector) và "What" (Hành động). Cấu trúc JSON này giúp robot có khả năng "tái lập" (replay) hành vi con người.
*   **Hệ thống phân cấp Worker:** Có các worker riêng biệt cho từng nhiệm vụ: `schedule-worker` (chạy theo lịch), `pgboss-worker` (xử lý ghi hình/thực thi), đảm bảo server chính không bị treo khi có nhiều tác vụ nặng đồng thời.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Kỹ thuật "Stealth" (Tàng hình):** Sử dụng `playwright-extra-plugin-stealth` và kỹ thuật tiêm dấu vân tay (fingerprint injection) để tránh bị các hệ thống chống bot (như Cloudflare) phát hiện.
*   **DOM Snapshots & Replay (rrweb):** Sử dụng thư viện `rrweb` để chụp lại trạng thái cây DOM và "phát lại" trên trình duyệt của người dùng. Kỹ thuật này giúp người dùng nhìn thấy trang web đang scrape ngay trên dashboard mà không cần stream video (tiết kiệm băng thông).
*   **Heuristic Data Detection:** Maxun sử dụng các thuật toán heuristic (trong `scraper.js`) để tự động nhận diện các danh sách (lists) và bảng dữ liệu dựa trên diện tích phần tử và mật độ văn bản.
*   **LLM-Vision Integration:** Kỹ thuật kết hợp chụp ảnh màn hình và mã HTML gửi cho AI để tự động sinh ra bộ chọn (selectors) mà không cần người dùng phải biết về CSS Selector hay XPath.
*   **Batch Persistence:** Dữ liệu trích xuất được lưu trữ theo lô (batches) vào PostgreSQL để tối ưu hóa hiệu năng ghi, tránh quá tải khi robot thu thập hàng vạn dòng dữ liệu.
*   **Model Context Protocol (MCP) Support:** Một kỹ thuật mới cho phép các Agent AI (như Claude Desktop) kết nối và sử dụng Maxun như một "kỹ năng" để đọc web.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của Maxun gồm 5 giai đoạn chính:

1.  **Khởi tạo (Initialization):** Người dùng nhập URL. Server cấp phát một slot trình duyệt từ `BrowserPool` và bắt đầu phiên ghi hình thông qua Socket.io.
2.  **Huấn luyện Robot (Training):** 
    *   *Recorder Mode:* Người dùng tương tác trực tiếp trên giao diện web. Hệ thống ghi lại các hành động và tự động tạo ra các bộ chọn (Selectors) tối ưu.
    *   *AI Mode:* Người dùng mô tả bằng văn bản -> AI phân tích cấu trang web -> Tự động tạo Workflow.
3.  **Lưu trữ Workflow:** Workflow được chuẩn hóa thành tệp JSON chứa các bước thực thi và lưu vào cơ sở dữ liệu cùng với MetaData (tên, lịch chạy, cấu hình proxy).
4.  **Thực thi (Execution):** 
    *   Robot chạy workflow thông qua `WorkflowInterpreter`. 
    *   Nếu trang web yêu cầu cuộn trang hoặc phân trang (pagination), robot sẽ thực hiện tự động cho đến khi đạt giới hạn (limit).
5.  **Xử lý đầu ra (Output Processing):** 
    *   Dữ liệu thô được chuyển đổi thành Markdown (cho AI), HTML hoặc JSON.
    *   Ảnh chụp màn hình được đẩy lên MinIO.
    *   Dữ liệu cuối cùng được đồng bộ hóa sang Google Sheets, Airtable hoặc gửi qua Webhook cho ứng dụng của bên thứ ba.