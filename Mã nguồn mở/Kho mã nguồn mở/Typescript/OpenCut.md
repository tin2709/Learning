Dựa trên mã nguồn và tài liệu của dự án **OpenCut**, đây là phân tích chuyên sâu về kiến trúc và kỹ thuật của một trình biên tập video hiện đại chạy trên nền tảng Web/Native.

### 1. Phân tích Công nghệ Cốt lõi (Core Technology Stack)

Khác với LosslessCut (dựa vào FFmpeg), OpenCut xây dựng một **Engine xử lý đồ họa riêng** dựa trên Rust và GPU:

*   **Rust & WGPU (Trái tim của Engine):** Sử dụng `wgpu` để trừu tượng hóa các API đồ họa (WebGPU, Vulkan, Metal, DX12). Điều này cho phép chạy cùng một mã nguồn render trên trình duyệt (WASM) và ứng dụng máy tính bản địa (Native).
*   **Next.js & TypeScript (Vỏ bọc Web):** Giao diện người dùng được xây dựng bằng React/Next.js, tận dụng các component từ Radix UI và Lucide icons.
*   **GPUI (Native Desktop):** Ứng dụng Desktop sử dụng framework GPUI (được phát triển bởi đội ngũ tạo ra Zed editor), giúp xây dựng giao diện hoàn toàn bằng GPU, cực kỳ mượt mà.
*   **Hệ thống lưu trữ lai (Hybrid Storage):** 
    *   **IndexedDB:** Lưu trữ metadata của dự án và dữ liệu JSON.
    *   **OPFS (Origin Private File System):** Lưu trữ file video/audio thô ngay trong trình duyệt với hiệu suất đọc/ghi gần như ổ đĩa thật.
*   **WASM AI (Whisper):** Tích hợp `@huggingface/transformers` chạy trong Web Worker để tự động tạo phụ đề từ giọng nói mà không cần gửi dữ liệu lên server (Privacy-first).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OpenCut được thiết kế để **"Rust-hóa" toàn bộ logic kinh doanh**:

*   **Logic-as-a-Crate:** Logic quan trọng nhất (Compositor, Effects, Masks, Time) được tách thành các crate Rust riêng biệt trong thư mục `rust/crates/`. TypeScript chỉ đóng vai trò là lớp "giao tiếp" và hiển thị UI.
*   **Hệ thống thời gian Ticks (120,000 TPS):** Để tránh sai số dấu phẩy động (floating-point error) khi làm việc với frame rate lẻ (23.976, 29.97), OpenCut sử dụng đơn vị "Ticks" nguyên số. 120,000 là con số "vàng" vì nó chia hết cho hầu hết các mẫu số của các frame rate phổ biến.
*   **Command Pattern & Command Manager:** Mọi thao tác trên timeline (cắt, di chuyển, thêm hiệu ứng) đều là một đối tượng `Command`. Điều này giúp quản lý Undo/Redo cực kỳ chặt chẽ và nhất quán.
*   **Ripple Editing Algorithm:** Thay vì tính toán thủ công vị trí các clip, hệ thống sử dụng thuật toán so sánh "Interval Sets" để tự động đẩy/lùi các clip trên timeline khi có sự thay đổi (Ripple).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Jump Flood Algorithm (JFA):** Trong phần xử lý mask (`rust/crates/masks/src/sdf.rs`), dự án sử dụng thuật toán JFA để tính toán Signed Distance Fields (SDF). Đây là kỹ thuật cực nhanh để tạo ra hiệu ứng làm mềm biên (feathering) cho mask trên GPU.
*   **Custom WASM Bridge:** File `rust/crates/bridge` chứa một procedural macro (`#[export]`) tự động chuyển đổi tên hàm từ `snake_case` (Rust) sang `camelCase` (JS) khi biên dịch sang WASM, giúp giảm thiểu lỗi gõ tên hàm thủ công.
*   **Recursive Descent Parser cho Math:** OpenCut cho phép người dùng nhập phép tính vào các ô số (ví dụ: `1920/2 + 10`). Họ đã tự viết một parser phân tích cú pháp để tính toán an toàn thay vì dùng hàm `eval()` nguy hiểm.
*   **Gaussian Blur Đa tầng (Multi-pass Blur):** Để tối ưu hiệu năng, hiệu ứng Blur được tách thành 2 pass (ngang và dọc), kết hợp với kỹ thuật `u_step` để mở rộng phạm vi mờ mà không làm tăng số lượng mẫu thử (samples).
*   **Strict Object Params (ESLint):** Dự án tự viết rule ESLint riêng (`prefer-object-params.mjs`) để bắt buộc các hàm có từ 2 tham số trở lên phải dùng kiểu đối tượng (`destructured object`), giúp mã nguồn dễ mở rộng và tự giải thích (self-documenting).

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Nạp Media (Ingest):** Video được kéo vào -> Đưa vào OPFS -> `mediabunny` phân tích codec/metadata -> Tạo thumbnail bằng canvas 2D.
2.  **Tương tác (Interaction):** Người dùng kéo clip -> `DragDropController` tính toán `DropTarget` -> Gửi Command tới `TimelineManager` -> Cập nhật trạng thái `Zustand`.
3.  **Xem trước (Preview):** 
    *   React cập nhật state.
    *   `RendererManager` xây dựng cây render (`RootNode`).
    *   WASM Compositor nạp texture (các khung hình video từ `videoCache`) và chạy shader WGSL.
    *   Kết quả hiển thị lên một thẻ `<canvas>` duy nhất.
4.  **Xuất bản (Export):** `SceneExporter` chạy một vòng lặp không đồng bộ, yêu cầu Renderer vẽ từng khung hình, sau đó dùng `VideoEncoder` (WebCodecs) để đóng gói thành file MP4/WebM ngay tại client.

### Đánh giá:
OpenCut không chỉ là một ứng dụng web; nó là một nỗ lực nghiêm túc trong việc đưa **hiệu năng đồ họa cấp hệ thống** lên trình duyệt. Việc di cư dần từ TypeScript sang Rust cho thấy tham vọng xây dựng một engine biên tập video chuyên nghiệp, bền vững và thực sự đa nền tảng.