Dưới đây là phân tích chi tiết về dự án **FieldTrip** dựa trên cấu trúc thư mục và mã nguồn bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ:** **TypeScript** là ngôn ngữ chủ đạo (61%), đảm bảo tính an toàn về kiểu dữ liệu (type-safe) cho các bộ parser phức tạp.
*   **CLI & Backend:** 
    *   **Commander:** Xử lý các tham số dòng lệnh.
    *   **Express:** Server siêu nhẹ để phục vụ API và giao diện UI tại máy cục bộ (localhost).
    *   **MiniSearch:** Công cụ tìm kiếm toàn văn (Full-text search) nhỏ gọn, hỗ trợ tìm kiếm mờ (fuzzy search) và tiền tố (prefix) mà không cần database rời.
*   **Parsers (Bộ phân giải):**
    *   **Protobufjs:** Phân tích các tệp `.proto`.
    *   **YAML:** Xử lý OpenAPI/AsyncAPI.
    *   **Glob:** Quét đệ quy các tệp tin trong thư mục.
*   **Frontend (UI Local):**
    *   **Vite:** Bundler cho giao diện người dùng.
    *   **D3.js:** Xử lý trực quan hóa dữ liệu dạng đồ thị (Graph view).
    *   **Monaco Editor:** Trình soạn thảo mã nguồn (giống VS Code) để xem schema với tính năng highlight cú pháp và định vị dòng.
*   **Website Landing:** **Astro** được dùng để xây dựng trang giới thiệu sản phẩm.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của FieldTrip đi theo mô hình **"Scan-Parse-Index-Serve"** (Quét - Phân giải - Đánh chỉ mục - Phục vụ), tối ưu cho trải nghiệm chạy ngay không cần cấu hình (`npx`):

*   **Tính chuẩn hóa (Normalization):** Mọi định dạng schema (Proto, Avro, OpenAPI...) sau khi qua bộ parser riêng biệt đều được đưa về một interface chung duy nhất là `SchemaProperty`. Điều này cho phép hệ thống tìm kiếm và trực quan hóa hoạt động thống nhất dù nguồn dữ liệu là gì.
*   **Kiến trúc Local-first (Ưu tiên cục bộ):** Công cụ không đẩy dữ liệu lên cloud. Nó tạo ra một môi trường khám phá tạm thời ngay trên máy người dùng, đảm bảo quyền riêng tư và tốc độ truy xuất tệp tin gốc.
*   **Tách biệt mối quan tâm (Separation of Concerns):**
    *   `src/scanner`: Chỉ lo việc tìm tệp.
    *   `src/parsers`: Chỉ lo việc đọc nội dung.
    *   `src/indexer`: Chỉ lo việc tối ưu tìm kiếm.
    *   `ui`: Chịu trách nhiệm hiển thị đa chế độ (Table, Matrix, Graph).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Heuristic Schema Detection (Nhận diện schema thông minh):** Trong `src/parsers/index.ts`, FieldTrip không chỉ dựa vào đuôi tệp (`.json`, `.yaml`) mà còn kiểm tra các "dấu hiệu" nội dung (như sự hiện diện của key `openapi`, `asyncapi`, `$schema`) để chọn bộ parser phù hợp.
*   **Đồ thị quan hệ (Force-Directed Graph):** Sử dụng D3.js để mô phỏng lực hấp dẫn giữa các nút. Kỹ thuật này giúp phát hiện các "bridge fields" (các trường dữ liệu dùng chung) nối giữa các dịch vụ khác nhau trong hệ thống microservices.
*   **Matrix Heatmap:** Kỹ thuật hiển thị ma trận để so sánh sự hiện diện của các trường (Required vs Optional) trên quy mô lớn, giúp kiến trúc sư dữ liệu phát hiện sự không nhất quán giữa các schema.
*   **Virtual Highlighting:** Khi người dùng click vào một trường trong UI, hệ thống gọi API để lấy nội dung tệp gốc và sử dụng Monaco Editor để cuộn (scroll) và highlight chính xác vị trí của trường đó thông qua việc phân tích regex dòng văn bản.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi động:** Người dùng chạy lệnh `fieldtrip --dir ./schemas`.
2.  **Quét (Scanner):** Tìm tất cả tệp có đuôi `.yaml, .json, .proto, .avsc` trừ các thư mục rác như `node_modules`.
3.  **Phân giải (Parsing):**
    *   Mỗi tệp được đưa vào bộ parser tương ứng.
    *   Hệ thống đệ quy (`walk`) qua các object, allOf/anyOf, nested messages để trích xuất tên trường, kiểu dữ liệu, mô tả và trạng thái `required`.
4.  **Đánh chỉ mục (Indexing):** MiniSearch nhận danh sách các thuộc tính và xây dựng bộ chỉ mục tìm kiếm trong bộ nhớ (RAM).
5.  **Phục vụ (Serving):** 
    *   Express server khởi động tại cổng 3200.
    *   Mở trình duyệt tự động (sử dụng gói `open`).
6.  **Tương tác (Exploring):**
    *   **Table View:** Tìm kiếm nhanh.
    *   **Matrix View:** So sánh độ phủ của các trường.
    *   **Graph View:** Xem cấu trúc kết nối giữa các tệp schema.
    *   **Detail View:** Xem code thực tế của schema ngay trong UI.

### Đánh giá:
Đây là một công cụ **Developer Experience (DX)** tuyệt vời cho các hệ thống lớn. Nó giải quyết bài toán "Schema Sprawl" (Sự bùng nổ schema) bằng cách biến các file text thô sơ thành một cơ sở dữ liệu có thể truy vấn và trực quan hóa được ngay lập tức.