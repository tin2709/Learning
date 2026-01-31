Dựa trên các tệp tin và mã nguồn bạn cung cấp cho dự án **BentoPDF**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của hệ thống này:

---

### 1. Công Nghệ Cốt Lõi (Core Technology Stack)

BentoPDF được xây dựng trên triết lý **"Privacy-First" (Quyền riêng tư là trên hết)**, tận dụng sức mạnh của trình duyệt hiện đại để xử lý dữ liệu mà không cần máy chủ.

*   **Ngôn ngữ & Build Tool:**
    *   **TypeScript:** Sử dụng xuyên suốt để đảm bảo kiểu dữ liệu mạnh (type-safety) và giảm lỗi logic.
    *   **Vite:** Công cụ build siêu nhanh, hỗ trợ Hot Module Replacement (HMR) và tối ưu hóa tài nguyên đầu ra.
*   **Xử lý PDF (Trái tim của hệ thống):**
    *   **PDF-lib.js:** Thư viện chính để thao tác trực tiếp với cấu trúc file PDF (gộp, tách, chỉnh sửa nội dung).
    *   **PDF.js (Mozilla):** Dùng để render PDF hiển thị trên trình duyệt.
    *   **PDFKit:** Dùng để tạo ra các tài liệu PDF mới từ dữ liệu.
    *   **WebAssembly (WASM):** Đây là công nghệ quan trọng nhất giúp chạy các phần mềm C/C++ nặng ngay trong trình duyệt:
        *   **LibreOffice WASM:** Chuyển đổi Word/Excel/PPT sang PDF.
        *   **qpdf-wasm:** Kiểm tra, sửa lỗi và chuyển đổi cấu trúc PDF nâng cao.
        *   **Tesseract.js:** Nhận dạng ký tự quang học (OCR).
        *   **OpenCV.js:** Xử lý hình ảnh (ví dụ: căn chỉnh tài liệu bị lệch - Deskew).
*   **Giao diện (UI/UX):**
    *   **Tailwind CSS:** Framework CSS tiện ích giúp xây dựng giao diện nhanh và đáp ứng (responsive).
    *   **Lucide & Phosphor Icons:** Bộ icon hiện đại.
    *   **Handlebars:** Dùng làm template engine để tái sử dụng các thành phần như Navbar, Footer.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của BentoPDF mang tính đột phá so với các công cụ PDF truyền thống:

*   **Kiến trúc Client-Side (Zero-Server Processing):** Mọi phép toán xử lý file đều diễn ra trong RAM của trình duyệt người dùng. Điều này loại bỏ hoàn toàn rủi ro rò rỉ dữ liệu lên server, đồng thời giảm chi phí vận hành server đáng kể.
*   **Thiết kế Dual-Licensing & Module tách biệt:**
    *   Hệ thống tách biệt rõ ràng giữa mã nguồn cốt lõi (AGPL-3.0) và các thư viện bên thứ ba có giấy phép khắt khe (như Ghostscript, PyMuPDF).
    *   Các thư viện này không được đóng gói sẵn mà được **nạp động (dynamic loading)** qua cấu hình URL (WASM Proxy) để đảm bảo tính pháp lý.
*   **Tư duy Docker hóa an toàn (Security-Hardened):**
    *   Sử dụng `nginx-unprivileged`: Container chạy với quyền user thấp (non-root), tăng cường khả năng chống tấn công leo thang đặc quyền.
    *   Hỗ trợ đa nền tảng (amd64, arm64) thông qua Docker build.
*   **Hỗ trợ Hosting Tĩnh (Static Hosting Ready):** Do không cần backend, dự án có thể chạy trên GitHub Pages, Netlify hoặc Vercel mà không tốn chi phí server.

---

### 3. Các Kỹ Thuật Chính (Key Technical Features)

*   **Web Workers (Đa luồng):** Các tác vụ nặng như gộp hàng trăm file hoặc OCR được đẩy vào Web Worker (chạy ngầm) để không làm treo giao diện người dùng (UI thread).
*   **Dynamic WASM Loading:** Chỉ tải các module WASM (thường rất nặng, hàng chục MB) khi người dùng thực sự sử dụng công cụ đó (ví dụ: chỉ tải module LibreOffice khi cần chuyển từ Word sang PDF).
*   **Tối ưu hóa bộ nhớ (Memory Management):** Sử dụng các flag như `NODE_OPTIONS='--max-old-space-size=3072'` trong quá trình build để xử lý các file i18n (đa ngôn ngữ) đồ sộ mà không bị tràn bộ nhớ (OOM).
*   **CORS Proxy via Cloudflare Workers:** Một kỹ thuật thông minh để giải quyết vấn đề CORS khi cần tải chứng chỉ số từ các máy chủ bên ngoài cho tính năng Chữ ký số (Digital Signature).
*   **PWA & Service Workers:** Hỗ trợ lưu trữ đệm (caching) các tài nguyên để ứng dụng có thể hoạt động ngoại tuyến (Offline mode).

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng đi của dữ liệu trong BentoPDF diễn ra như sau:

1.  **Giai đoạn Nhập (Input):**
    *   Người dùng chọn file từ máy tính (thông qua `fileHandler.ts`).
    *   Dữ liệu được đọc dưới dạng `ArrayBuffer` hoặc `Blob` trong trình duyệt.
2.  **Giai đoạn Khởi tạo (Initialization):**
    *   Hệ thống xác định công cụ người dùng yêu cầu (ví dụ: gộp file).
    *   Nạp thư viện JS tương ứng hoặc tải module WASM cần thiết (nếu chưa có trong cache).
3.  **Giai đoạn Xử lý (Processing - Trong Web Worker):**
    *   File được gửi vào Worker.
    *   Thư viện (như `pdf-lib`) thực hiện thao tác sửa đổi cấu trúc file trên RAM.
    *   Nếu là chuyển đổi định dạng, module WASM (như LibreOffice) sẽ nhận dữ liệu ảo và xuất ra file PDF.
4.  **Giai đoạn Xuất (Output):**
    *   Dữ liệu sau xử lý được chuyển ngược lại luồng chính (Main thread).
    *   Tạo ra một `Blob URL` tạm thời.
    *   Kích hoạt trình duyệt tải file về máy người dùng hoặc hiển thị lên `PDF Viewer`.
5.  **Giai đoạn Giải phóng:**
    *   Dữ liệu tạm trong RAM được xóa sạch khi kết thúc phiên làm việc để bảo mật.

### Tổng kết
BentoPDF là một ví dụ điển hình của việc đưa sức mạnh ứng dụng desktop lên web thông qua **WebAssembly** và **TypeScript**, giải quyết triệt để bài toán bảo mật dữ liệu nhạy cảm trong tài liệu PDF.