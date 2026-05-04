Dựa trên tài liệu và cấu trúc mã nguồn của dự án **Grida**, dưới đây là phân tích chuyên sâu về Công nghệ cốt lõi, Tư duy Kiến trúc, Kỹ thuật lập trình và Luồng hoạt động của hệ thống.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Grida không chỉ là một ứng dụng web mà là một **Graphics Engine** đa nền tảng.

*   **Ngôn ngữ lập trình:**
    *   **Rust:** Sử dụng cho lõi xử lý đồ họa (Core Engine), tính toán toán học phức tạp và xử lý font/CSS. Rust đảm bảo hiệu suất tối đa và an toàn bộ nhớ.
    *   **TypeScript:** Sử dụng cho tầng giao diện (Editor), SDK và các công cụ tích hợp.
*   **Rendering Engine (Skia & WebAssembly):**
    *   Sử dụng **Skia** (thư viện đồ họa 2D đứng sau Chrome/Flutter) thông qua bindings `skia-safe`.
    *   Biên dịch sang **WebAssembly (WASM)** bằng Emscripten để chạy mượt mà trên trình duyệt với hiệu suất gần như ứng dụng bản địa (native).
*   **Định dạng dữ liệu (FlatBuffers):**
    *   Thay vì JSON, Grida sử dụng **FlatBuffers** (`.grida` format). Công nghệ này cho phép truy cập dữ liệu mà không cần giải tuần tự hóa (parsing), cực kỳ quan trọng cho các file thiết kế lớn chứa hàng chục nghìn node.
*   **Tầng dữ liệu & Backend:**
    *   **Supabase (PostgreSQL):** Quản lý cơ sở dữ liệu, Auth và Storage.
    *   **Cloudflare Workers:** Sử dụng cho các tác vụ thời gian thực và đồng bộ hóa tài liệu (Document Worker).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Grida được xây dựng dựa trên sự phân tách triệt để giữa **Dữ liệu - Logic - Hiển thị**.

*   **Kiến trúc Monorepo:** Quản lý bằng `pnpm workspaces` và `turbo`. Phân tách rõ ràng:
    *   `crates/`: Logic lõi bằng Rust (cascade CSS, math, font parser).
    *   `packages/`: Các module dùng chung (sync, color, schema).
    *   `apps/`: Các ứng dụng đầu cuối (editor, docs, blog).
*   **Hybrid Rendering Strategy:** Đây là điểm độc đáo. Grida hỗ trợ hai backend:
    *   **DOM Backend:** Dùng cho các luồng công việc làm website (Website Builder), tận dụng khả năng render HTML/CSS tự nhiên của trình duyệt.
    *   **Skia/WASM Backend:** Dùng cho Editor chuyên nghiệp, đòi hỏi độ chính xác đến từng pixel và hiệu suất canvas cực cao.
*   **Headless-First Design:** Engine đồ họa được thiết kế để có thể chạy không cần giao diện (Headless). Package `@grida/refig` cho phép render file Figma trực tiếp trên Node.js hoặc CI/CD pipeline để xuất ảnh/PDF mà không cần mở trình duyệt.
*   **Intermediate Representation (IR):** Hệ thống chuyển đổi mọi định dạng đầu vào (Figma, SVG, HTML) về một dạng biểu diễn trung gian thống nhất trước khi lưu trữ hoặc render.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Tối ưu hóa WASM (Binary Size vs Performance):**
    *   Dự án áp dụng kỹ thuật tinh vi: Dùng `opt-level = 3` cho Core Engine (ưu tiên tốc độ) và `opt-level = "z"` cho các parser (ưu tiên giảm kích thước file `.wasm`). Điều này giúp giảm ~15% dung lượng bundle mà không ảnh hưởng tốc độ thao tác của người dùng.
*   **FFI & Memory Bridge:**
    *   Kỹ thuật truyền dữ liệu qua lại giữa JavaScript và Rust/WASM thông qua bộ nhớ chia sẻ (Shared Memory), hạn chế tối đa việc sao chép dữ liệu (Zero-copy).
*   **Stylo Integration:**
    *   Tích hợp **Stylo** (engine CSS của Firefox) vào Rust để xử lý cascade CSS một cách chính xác nhất mà không cần phụ thuộc vào engine của trình duyệt đang chạy.
*   **Reducer-based State Management:**
    *   Sử dụng mô hình Reducer (tương tự Redux/XState) kết hợp với thư viện `immer` để quản lý các trạng thái phức tạp của Canvas, đảm bảo tính bất biến (immutability) và khả năng Undo/Redo mạnh mẽ.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Nhập (Import Flow):
1.  **Input:** Người dùng kéo file `.fig` hoặc dán link Figma/SVG.
2.  **Parser:** `@grida/io-figma` hoặc `@grida/io-svg` phân tích cấu trúc node.
3.  **Normalization:** Chuyển đổi các thuộc tính đặc thù của Figma về Grida Schema (FlatBuffers).
4.  **Persistence:** Lưu trữ metadata vào Supabase và file binary vào Storage.

#### B. Luồng Biên tập (Editing Flow):
1.  **Interaction:** Người dùng di chuyển một đối tượng trên Canvas.
2.  **Action:** JavaScript gửi tọa độ/thao tác xuống lõi Rust thông qua WASM.
3.  **Calculation:** Rust thực hiện tính toán toán học (snapping, biến đổi affine) và cập nhật cây FlatBuffers.
4.  **Re-render:** Skia vẽ lại khung hình (60 FPS) dựa trên trạng thái mới.
5.  **Sync:** Đồng bộ thay đổi lên Cloud thông qua WebSockets/Durable Objects.

#### C. Luồng Xuất (Export Flow):
1.  **Request:** Yêu cầu xuất ảnh PNG ở scale 2x.
2.  **Processing:** Lõi Rust tạo một Raster Surface ẩn trong bộ nhớ.
3.  **Baking:** Vẽ toàn bộ cây node lên surface đó.
4.  **Encoding:** Mã hóa thành byte ảnh và gửi ngược lại cho JavaScript để tải về.

---

### Tổng kết
Grida không chỉ dừng lại ở một công cụ thiết kế, nó là một nỗ lực tái cấu trúc cách chúng ta xử lý đồ họa trên Web. Bằng cách kết hợp **an toàn của Rust**, **sức mạnh của Skia** và **tính linh hoạt của WASM**, dự án tạo ra một nền tảng có khả năng thay thế các công cụ đóng như Figma, đồng thời mở rộng khả năng lập trình cho các nhà phát triển đồ họa.