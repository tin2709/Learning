Chào bạn, đây là bản phân tích chi tiết về dự án **TaxHacker**, một ứng dụng kế toán cá nhân/doanh nghiệp nhỏ dựa trên AI, được xây dựng bằng các công nghệ hiện đại nhất trong hệ sinh thái JavaScript/TypeScript.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án này là một minh chứng cho xu hướng "AI-Native App" hiện nay:

*   **Framework chính:** **Next.js 15+** (App Router). Sử dụng tối đa tính năng mới như **React Server Components (RSC)** và **Server Actions** để xử lý logic backend ngay trong component.
*   **Ngôn ngữ:** **TypeScript (99.2%)** - Đảm bảo tính chặt chẽ cho dữ liệu tài chính.
*   **Cơ sở dữ liệu & ORM:** **PostgreSQL** đi kèm với **Prisma**. Đây là lựa chọn tiêu chuẩn cho các ứng dụng web cần quan hệ dữ liệu phức tạp.
*   **Trí tuệ nhân tạo (AI):** 
    *   **LangChain:** Sử dụng để trừu tượng hóa các lời gọi LLM (OpenAI, Gemini, Mistral).
    *   **LLM Vision:** Tận dụng khả năng đọc hiểu hình ảnh của GPT-4o hoặc Gemini để trích xuất dữ liệu từ hóa đơn.
*   **Xử lý hình ảnh/Tài liệu:**
    *   **Ghostscript & GraphicsMagick:** Dùng để xử lý file PDF.
    *   **Sharp & pdf2pic:** Chuyển đổi PDF/Ảnh sang định dạng mà AI có thể "nhìn" thấy (Vision).
*   **Xác thực:** **Better-Auth** – Một thư viện auth mới nổi, mạnh mẽ và linh hoạt.
*   **Giao diện:** **Tailwind CSS + Shadcn/UI** – Giúp giao diện sạch sẽ, chuyên nghiệp như một ứng dụng SaaS hiện đại.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án được thiết kế với tư duy **"Self-hosted first, Cloud second"**:

*   **Hybrid Deployment:** Kiến trúc hỗ trợ cả hai chế độ. Chế độ `SELF_HOSTED_MODE` cho phép người dùng tự quản lý API Key của LLM và lưu trữ file cục bộ, đảm bảo tính riêng tư tuyệt đối.
*   **State-based Document Management:** Quy trình xử lý file được chia làm 2 trạng thái rõ ràng:
    *   **Unsorted:** File vừa upload, chưa được AI phân tích hoặc chưa gán vào giao dịch.
    *   **Transactions:** Dữ liệu đã được cấu trúc hóa và lưu vào sổ cái.
*   **Strategy Pattern cho LLM:** Trong `ai/providers/llmProvider.ts`, kiến trúc sử dụng pattern này để thống nhất đầu vào/đầu ra của các nhà cung cấp AI khác nhau, giúp việc đổi từ OpenAI sang Gemini chỉ là thay đổi config.
*   **Data Portability (Tính di động của dữ liệu):** Dự án rất chú trọng vào việc Backup/Export. Người dùng có thể tải toàn bộ file + database (dưới dạng JSON) để chuyển nhà cung cấp, đúng tinh thần Open Source.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Structured Output từ LLM:** Sử dụng tính năng `withStructuredOutput` của LangChain để ép AI trả về dữ liệu đúng schema (JSON) thay vì văn bản tự do. Điều này giúp code có thể lưu trực tiếp vào DB mà không cần parse thủ công phức tạp.
*   **Dynamic Prompt Building:** Trong `ai/prompt.ts`, hệ thống tự động build prompt dựa trên các `Field` (trường tùy chỉnh) mà người dùng định nghĩa. Đây là kỹ thuật giúp ứng dụng cực kỳ linh hoạt (người dùng tự tạo cột trong "Excel", AI tự biết cách tìm dữ liệu cho cột đó).
*   **Event-Stream for Progress:** Sử dụng Server-Sent Events (SSE) trong `api/progress` để cập nhật tiến độ thực hiện các tác vụ nặng (như export hàng nghìn file) lên giao diện theo thời gian thực.
*   **Poor Man's Cache:** Triển khai một cơ chế cache đơn giản trong `api/currency/route.ts` để lưu tỷ giá hối đoái, giảm số lượng request tới các dịch vụ bên thứ ba (xe.com).
*   **Zod Validation:** Sử dụng Zod xuyên suốt từ việc validate Form cho đến kết quả trả về từ AI, đảm bảo dữ liệu luôn đúng định dạng trước khi chạm vào Database.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình từ lúc có một tờ hóa đơn giấy đến khi lên báo cáo:

1.  **Giai đoạn Thu thập:** Người dùng chụp ảnh/upload PDF. File được lưu vào thư mục `UPLOAD_PATH` theo cấu trúc ID người dùng.
2.  **Giai đoạn Tiền xử lý:** Nếu là PDF, hệ thống dùng `pdf2pic` để tạo ảnh xem trước (Preview). Ảnh này sẽ được gửi tới AI.
3.  **Giai đoạn Phân tích (AI Analysis):** 
    *   Hệ thống lấy danh sách `Field` và `Category` của người dùng.
    *   Build một prompt khổng lồ gửi kèm ảnh tới LLM Vision.
    *   AI trả về JSON chứa: Tên cửa hàng, ngày, tổng tiền, loại tiền tệ, các mặt hàng lẻ (Line items).
4.  **Giai đoạn Chuyển đổi (Currency Conversion):** Hệ thống lấy ngày của giao dịch, gọi API để lấy tỷ giá lịch sử và quy đổi về đơn vị tiền tệ cơ sở của người dùng.
5.  **Giai đoạn Hậu xử lý (Item Splitting):** Nếu hóa đơn có nhiều mục (ví dụ đi siêu thị mua cả đồ ăn và đồ dùng), người dùng có thể dùng công cụ `ItemsDetectTool` để tách một hóa đơn thành nhiều giao dịch riêng biệt.
6.  **Giai đoạn Lưu trữ & Xuất bản:** Dữ liệu được xác nhận và lưu vào PostgreSQL. Khi cần, hệ thống dùng `jszip` và `fast-csv` để đóng gói toàn bộ ảnh và dữ liệu thành một file ZIP cho kế toán.

---

### 5. Đánh giá tổng quan

**TaxHacker** là một dự án "thực dụng" cực cao. Nó không chỉ là một wrapper cho ChatGPT mà là một hệ thống quản lý tài liệu hoàn chỉnh. Kỹ thuật lập trình ở đây rất hiện đại (Next.js 15, Better-Auth, LangChain), cho thấy tác giả là người nắm bắt công nghệ rất nhanh và có tư duy sản phẩm tốt. Dự án này rất phù hợp để học cách xử lý file, làm việc với AI Vision và xây dựng các hệ thống tự triển khai (self-hosted).