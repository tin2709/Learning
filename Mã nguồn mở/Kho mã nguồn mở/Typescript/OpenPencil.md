Dựa trên cấu trúc thư mục và nội dung các file chiến lược (`README.md`, `AGENTS.md`, `package.json`, và các file cấu hình), dưới đây là phân tích chuyên sâu về dự án **OpenPencil**:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

OpenPencil sử dụng một tổ hợp công nghệ hiện đại, tập trung vào hiệu năng đồ họa và khả năng mở rộng:

*   **Rendering Engine (Skia/CanvasKit WASM):** Đây là "trái tim" của hệ thống. Bằng cách sử dụng **CanvasKit** (Skia được biên dịch sang WebAssembly), OpenPencil đạt được hiệu năng vẽ tương đương với Figma. Nó không vẽ bằng HTML/CSS thông thường mà vẽ trực tiếp lên một bề mặt WebGL, cho phép xử lý hàng nghìn layer mượt mà.
*   **Layout Engine (Yoga WASM - Grid Fork):** Sử dụng Yoga (của Meta) phiên bản hỗ trợ **CSS Grid**. Điều này cho phép tính toán Auto Layout và Complex Grid cực nhanh trong môi trường WASM trước khi chuyển dữ liệu sang bộ lọc vẽ.
*   **Data Structure & Sync (Yjs & CRDT):** Để hỗ trợ cộng tác thời gian thực (Real-time Collaboration) mà không cần server trung tâm, dự án dùng **Yjs**. Dữ liệu được đồng bộ qua **WebRTC (Trystero)**, giúp các tệp thiết kế có tính chất "P2P" - dữ liệu đi trực tiếp giữa các trình duyệt.
*   **File Format & Codec (Kiwi Binary):** Dự án giải mã định dạng `.fig` của Figma bằng **Kiwi binary codec** kết hợp với **Zstd compression**. Đây là kỹ thuật kỹ thuật đảo ngược (reverse-engineering) cao cấp để đảm bảo tính tương thích.
*   **Runtime & Tooling:** Sử dụng **Bun** làm runtime chính cho tốc độ thực thi vượt trội so với Node.js, đặc biệt là trong các tác vụ CLI và xử lý file nặng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OpenPencil đi theo triết lý **"Headless-First"** và **"Programmable SDK"**:

*   **Monorepo Tách biệt Tuyệt đối:**
    *   `@open-pencil/core`: Chứa logic thuần túy (Scene graph, renderer, layout). Không phụ thuộc vào DOM, có thể chạy trong môi trường Server/CLI.
    *   `@open-pencil/vue`: Một Headless SDK. Nó cung cấp các logic UI (composables) nhưng không áp đặt giao diện, cho phép các nhà phát triển khác xây dựng trình chỉnh sửa tùy chỉnh (Custom Editor) dựa trên OpenPencil.
    *   `@open-pencil/cli` & `@open-pencil/mcp`: Tận dụng `@open-pencil/core` để cung cấp khả năng tự động hóa và tích hợp AI.
*   **Hệ thống Tool Registry thống nhất:** Mọi hành động chỉnh sửa (tạo hình, đổi màu, layout) đều được định nghĩa thành các `ToolDef`. Một công cụ (Tool) sau khi định nghĩa sẽ tự động khả dụng ở 3 nơi: Giao diện người dùng, Câu lệnh CLI, và AI Agent (thông qua MCP).
*   **Figma API Compatibility:** Dự án xây dựng một lớp Proxy (`FigmaAPI`) giả lập hoàn toàn API của Figma. Điều này cực kỳ thông minh vì nó cho phép các script/plugins hiện có của Figma có thể chạy trên OpenPencil với rất ít thay đổi.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Coding Techniques)

*   **Proxy Pattern cho Scene Graph:** Sử dụng JavaScript Proxies để ánh xạ các thuộc tính nội bộ của SceneNode sang định dạng thuộc tính của Figma API. Điều này giúp code gọn gàng và duy trì tính tương thích ngược.
*   **Optimized Rendering Loop:** Phân biệt rõ giữa `renderVersion` (chỉ vẽ lại khi zoom/pan - rẻ) và `sceneVersion` (vẽ lại khi dữ liệu thay đổi - đắt). Kỹ thuật này giúp tối ưu hóa FPS khi người dùng chỉ thao tác điều hướng canvas.
*   **Strict Linting & Quality Gate:** Sử dụng **Oxlint** với các luật tùy chỉnh khắt khe (file `lint/plugin.js`). Ví dụ:
    *   Cấm `Math.random()` (buộc dùng `crypto.getRandomValues` để đảm bảo tính duy nhất trong collab).
    *   Cấm `any` và các kiểu dữ liệu inline nếu đã có kiểu định nghĩa sẵn (như `Color`, `Vector`).
    *   Kiểm soát số lượng dòng code trong các `Composition Root` (không quá 260 dòng) để tránh "God Functions".
*   **Kiến trúc Event Bus (Nanoevents):** Thay vì dùng hệ thống event của Vue/React, dự án dùng một bus sự kiện nhẹ (`nanoevents`) trong lõi `core` để thông báo các thay đổi đồ thị (node updated, deleted) một cách nhanh chóng và hiệu quả.

---

### 4. Luồng Hoạt động Hệ thống (System Flow)

1.  **Giai đoạn Khởi tạo (Bootstrap):**
    *   Tải CanvasKit WASM và Yoga WASM.
    *   Khởi tạo `EditorContext` và `SceneGraph`.
    *   Nếu là Collab, thiết lập kết nối WebRTC qua Trystero và đồng bộ trạng thái Yjs.

2.  **Luồng Xử lý Dữ liệu (Data Pipeline):**
    *   **Input:** File `.fig` được đọc -> Giải mã Kiwi -> Convert thành `SceneNode` nội bộ -> Chạy `computeAllLayouts` (Yoga).
    *   **Mutation:** Người dùng/AI thực hiện thay đổi -> `EditorActions` gọi `SceneGraph` -> Ghi lại Snapshot cho Undo/Redo -> Emit event qua Bus.
    *   **Output:** `SkiaRenderer` lắng nghe event -> Thực hiện `render()` -> Flush dữ liệu ra GPU WebGL.

3.  **Luồng AI & MCP (AI Integration):**
    *   AI gửi yêu cầu qua giao diện Chat hoặc giao thức MCP.
    *   Yêu cầu được ánh xạ vào `ToolRegistry`.
    *   Hệ thống thực thi câu lệnh thông qua `FigmaAPI` proxy để thay đổi trực tiếp SceneGraph như một lập trình viên thực thụ.

### Tổng kết
OpenPencil không chỉ là một ứng dụng vẽ, mà là một **Nền tảng Thiết kế có khả năng lập trình (Programmable Design Platform)**. Kiến trúc monorepo cực kỳ chặt chẽ, việc tách biệt layer Rendering/Layout khỏi UI, và tư duy ưu tiên AI/Automation là những điểm sáng giúp nó trở thành một đối thủ mã nguồn mở tiềm năng của Figma.