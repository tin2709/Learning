Dưới đây là bản phân tích kỹ thuật chi tiết về dự án **ZIZIYI Office** – một giải pháp văn phòng "local-first" chạy hoàn toàn trên trình duyệt dựa trên tài liệu và mã nguồn bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

ZIZIYI Office thực chất là một bản phân phối lại của **OnlyOffice** được tinh chỉnh để chạy **Serverless** (không cần máy chủ điều phối).

*   **Framework chính:** **Next.js 15+** và **React 19**.
*   **Engine xử lý tài liệu:** **OnlyOffice SDKJS**. Đây là bộ thư viện cực kỳ mạnh mẽ xử lý việc render nội dung Word, Excel, PPT.
*   **Chuyển đổi định dạng (The Magic):** **OnlyOffice x2t WASM**. Dự án sử dụng WebAssembly để đưa mã nguồn C++ của OnlyOffice (vốn dùng để chuyển đổi file `.docx`, `.xlsx` sang định dạng JSON nội bộ) chạy trực tiếp trong trình duyệt.
*   **Quản lý trạng thái:** **Zustand** (để lưu cấu hình giao diện, ngôn ngữ, theme).
*   **Lưu trữ cục bộ:** **IndexedDB** (thông qua thư viện `idb`). Dự án sử dụng **File System Access API** để lưu các `FileSystemFileHandle`, cho phép người dùng mở lại các file cũ mà không cần upload lại.
*   **Xử lý đa luồng:** **Web Workers**. Các tác vụ nặng như chuyển đổi tài liệu bằng WASM được đẩy vào `x2t.worker.ts` để không làm treo giao diện người dùng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ZIZIYI Office rất độc đáo ở chỗ nó "đánh lừa" bộ Editor của OnlyOffice rằng nó đang kết nối với một máy chủ thực sự.

*   **Virtual Document Server:** Thông thường, OnlyOffice Editor yêu cầu một máy chủ (Document Server) để lưu trữ và phối hợp chỉnh sửa. ZIZIYI xây dựng lớp `EditorServer` (trong `utils/editor/server.ts`) đóng vai trò như một **máy chủ ảo** chạy ngay trong Main Thread.
*   **Cơ chế Intercept (Đánh chặn):** Đây là tư duy cốt lõi. OnlyOffice Editor giao tiếp qua HTTP (XHR/Fetch) và WebSocket. Dự án sử dụng kỹ thuật **Proxying**:
    *   Ghi đè `XMLHttpRequest` và `fetch` của trình duyệt bên trong Iframe của Editor.
    *   Tất cả các yêu cầu gửi đến "máy chủ" sẽ bị lớp Proxy chặn lại và chuyển hướng về lớp `EditorServer` cục bộ.
*   **Mocking WebSocket:** Sử dụng `MockSocket` để giả lập kết nối thời gian thực. Vì chỉ có một người dùng chỉnh sửa (local-first), các tín hiệu WebSocket được phản hồi ngay lập tức để Editor tin rằng nó đã kết nối thành công.
*   **Iframe Isolation:** Mỗi trình biên tập được nạp vào một Iframe riêng biệt để tránh xung đột biến toàn cục và CSS giữa các loại tài liệu khác nhau.

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **XHR & Fetch Proxying (`xhr.ts`, `fetch.ts`):** 
    *   Tạo ra một lớp bao bọc (Wrapper) cho các API mạng chuẩn.
    *   Sử dụng Middleware pattern để kiểm tra URL: Nếu URL liên quan đến tài liệu, nó sẽ lấy dữ liệu từ bộ nhớ RAM hoặc IndexedDB thay vì gửi request ra internet.
*   **WASM Integration & File Piping:** 
    *   Kỹ thuật nạp file `.wasm` lớn (như `x2t.wasm`) và quản lý bộ nhớ thông qua `FS` (FileSystem ảo của Emscripten).
    *   Dữ liệu được chuyển đổi qua lại giữa `Uint8Array` và `ArrayBuffer` để tối ưu hiệu suất xử lý binary.
*   **Zero-Copy Transferable Objects:** Trong `x2t.ts`, khi gửi dữ liệu giữa Main Thread và Web Worker, dự án sử dụng `transferables` để chuyển quyền sở hữu bộ nhớ thay vì copy dữ liệu, giúp xử lý các file Office hàng chục MB một cách tức thì.
*   **Hydration Handling trong Next.js:** Sử dụng `useHasHydrated` để đảm bảo các truy cập vào `localStorage` hoặc `IndexedDB` chỉ diễn ra sau khi Client-side rendering hoàn tất, tránh lỗi bất đồng bộ của SSR.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

#### Luồng 1: Mở một tài liệu hiện có
1.  Người dùng chọn file từ máy tính.
2.  Hệ thống lấy `FileSystemFileHandle` và lưu vào IndexedDB (để hiện trong mục "Recent").
3.  File binary được đưa vào **Web Worker**.
4.  **x2t WASM** biên dịch file (ví dụ `.docx`) thành định dạng `Editor.bin` (định dạng nhị phân nội bộ của OnlyOffice).
5.  `EditorServer` lưu `Editor.bin` vào một Map nội bộ và tạo một `Blob URL`.
6.  OnlyOffice Editor khởi động, gửi request lấy file, lớp Proxy chặn lại và trả về dữ liệu từ Map trên.

#### Luồng 2: Lưu tài liệu
1.  Người dùng nhấn nút Lưu hoặc hệ thống tự động lưu.
2.  Editor gửi dữ liệu chỉnh sửa (dưới dạng các gói tin nhị phân) qua XHR Proxy.
3.  `EditorServer` thu thập các mảnh (parts) dữ liệu này.
4.  Khi quá trình thu thập hoàn tất, **x2t WASM** được gọi để thực hiện luồng ngược lại: Chuyển từ định dạng nội bộ sang định dạng Office chuẩn (`.docx`, `.xlsx`, `.pptx`).
5.  Dữ liệu sau khi chuyển đổi được trả về Main Thread và kích hoạt trình tải xuống của trình duyệt (hoặc ghi đè vào file gốc nếu có quyền).

---

### Tổng kết
**ZIZIYI Office** là một ví dụ xuất sắc về việc tận dụng sức mạnh của trình duyệt hiện đại (WASM, Web Workers, File System API) để "phi tập trung hóa" các ứng dụng văn phòng nặng nề. Thay vì xây dựng lại từ đầu, dự án tập trung vào việc **xây dựng một lớp trung gian (Virtual Middleware)** để biến một Engine phía Server (OnlyOffice) thành một ứng dụng thuần Client-side.