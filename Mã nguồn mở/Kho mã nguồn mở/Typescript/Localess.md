Dựa trên kho lưu trữ và mã nguồn của dự án **Localess**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technology)

Localess được xây dựng trên mô hình **Serverless** hiện đại, tận dụng tối đa hệ sinh thái của Google Cloud và Firebase:

*   **Frontend:**
    *   **Angular (v20+):** Sử dụng các tính năng mới nhất như *Signals* (`@ngrx/signals`) để quản lý state và *Standalone Components*.
    *   **TailwindCSS (v4):** Framework CSS để xử lý giao diện nhanh và tối ưu.
    *   **Angular Material:** Cung cấp bộ UI component chuẩn chỉnh cho phần quản trị.
    *   **Tiptap/ProseMirror:** Dùng cho trình soạn thảo văn bản giàu tính năng (Rich Text Editor).
*   **Backend (BaaS - Backend as a Service):**
    *   **Firebase Authentication:** Quản lý định danh người dùng.
    *   **Cloud Firestore:** Cơ sở dữ liệu NoSQL dạng tài liệu để lưu trữ Schema, Nội dung và Bản dịch.
    *   **Cloud Storage:** Lưu trữ tài sản (Assets - hình ảnh, video) và các tệp JSON được xuất bản.
    *   **Cloud Functions (Node.js 22):** Xử lý logic nghiệp vụ nặng (AI Translation, Export/Import, Image Processing).
*   **AI & Công cụ tích hợp:**
    *   **Google Cloud Translation API & DeepL:** Tích hợp để tự động dịch thuật.
    *   **Unsplash API:** Cho phép tìm kiếm ảnh chất lượng cao trực tiếp từ UI.
    *   **Sharp & Exiftool-vendored:** Xử lý hình ảnh và trích xuất metadata trong Cloud Functions.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Localess thể hiện tư duy **"Headless CMS"** kết hợp với **"Low-code"**:

*   **Kiến trúc hướng Schema (Schema-Driven):** Thay vì tạo các bảng cứng nhắc, Localess cho phép người dùng định nghĩa "Schema" (cấu trúc dữ liệu). Tư duy này giúp hệ thống cực kỳ linh hoạt, có thể quản lý từ bài viết blog đến cấu hình ứng dụng phức tạp.
*   **Tách biệt Admin UI và Public API:**
    *   **Admin UI:** Giao tiếp trực tiếp với Firestore qua Firebase SDK để quản lý dữ liệu thời gian thực.
    *   **Public API (v1.ts):** Một Express app chạy trên Cloud Functions. Thay vì truy vấn trực tiếp Firestore (đắt và chậm), nó đọc các tệp JSON đã được "tĩnh hóa" trong Cloud Storage.
*   **Tư duy Tĩnh hóa (Static Site Generation logic):** Khi người dùng nhấn "Publish", hệ thống thực hiện chuyển đổi dữ liệu từ Firestore thành các tệp JSON tĩnh. Điều này giúp tăng tốc độ phản hồi (khoảng 20ms qua CDN) và giảm chi phí vận hành.
*   **Zod Validation:** Sử dụng Zod ở phía Backend (Functions) để đảm bảo dữ liệu Import/Export luôn đúng cấu trúc, tránh lỗi runtime.

---

### 3. Các kỹ thuật chính nổi bật

*   **Visual Editor (Real-time Preview):** File `sync-v1.ts` cho thấy kỹ thuật sử dụng `postMessage` để giao tiếp giữa ứng dụng của người dùng (trong iframe) và trang quản trị Localess. Điều này cho phép người dùng click vào nội dung trên web và nó sẽ tự động mở đúng trường dữ liệu đó trong CMS để chỉnh sửa.
*   **AI Automation:** Khi tạo một bản dịch mới, nếu tính năng `autoTranslate` được bật, Cloud Functions sẽ tự động gọi API dịch cho tất cả các ngôn ngữ mục tiêu dựa trên ngôn ngữ mặc định.
*   **Hệ thống Task không đồng bộ:** Các tác vụ nặng như Import/Export hàng nghìn tài liệu được xử lý qua `tasks.ts`. Nó sử dụng hàng đợi và cập nhật trạng thái (`INITIATED` -> `IN_PROGRESS` -> `FINISHED`) để người dùng theo dõi mà không làm treo UI.
*   **Xử lý Media thông minh:** Tự động tạo thumbnail cho video, resize ảnh theo tham số URL (ví dụ: `?w=300`) thông qua thư viện Sharp trong Cloud Functions.
*   **Versioning qua Cache Generation:** Sử dụng số `generation` của tệp tin trên Cloud Storage làm phiên bản cache (`cv`). Khi dữ liệu thay đổi, một tệp `cache.json` được cập nhật để thông báo cho Client tải dữ liệu mới nhất.

---

### 4. Tóm tắt luồng hoạt động (Workflow)

Quy trình từ lúc tạo dữ liệu đến khi ứng dụng người dùng nhận được nội dung:

1.  **Thiết lập (Setup):** Người dùng định nghĩa **Spaces** (Không gian làm việc) và các **Locales** (Ngôn ngữ).
2.  **Định nghĩa cấu trúc (Modeling):** Tạo **Schemas** để quy định các trường dữ liệu (Text, Number, Asset, Rich Text, v.v.).
3.  **Nhập liệu (Content Authoring):**
    *   Người dùng tạo **Translations** (cho i18n).
    *   Người dùng tạo **Contents** dựa trên Schemas đã định nghĩa.
    *   Tài nguyên (ảnh/video) được tải lên **Assets**.
4.  **Xuất bản (Publishing):**
    *   Khi nhấn "Publish", một Cloud Function được kích hoạt.
    *   Nó lấy dữ liệu từ Firestore, kết hợp với Schema để tạo ra tệp JSON hoàn chỉnh.
    *   Tệp JSON được lưu vào Cloud Storage theo cấu trúc: `spaces/{id}/contents/{documentId}/{locale}.json`.
5.  **Phân phối (Distribution):**
    *   Ứng dụng phía Client (Mobile/Web) gọi API V1: `/api/v1/spaces/{id}/contents/slugs/{path}`.
    *   API trả về dữ liệu từ Cloud Storage (được cache qua Google CDN).
6.  **Cập nhật:** Khi dữ liệu thay đổi, luồng Publish được lặp lại, cập nhật tệp tĩnh và làm mới cache.

**Kết luận:** Localess là một dự án có kỹ thuật rất cao, kết hợp khéo léo giữa sự tiện lợi của Firebase và sức mạnh của kiến trúc Headless CMS hiện đại. Nó giải quyết bài toán hiệu năng của Firebase bằng cách sử dụng "Static JSON Publishing".