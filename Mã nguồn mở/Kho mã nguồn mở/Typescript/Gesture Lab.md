Chào bạn, đây là bản phân tích chi tiết về dự án **Gesture Lab** dựa trên mã nguồn và tài liệu kiến trúc bạn cung cấp. Đây là một dự án thực nghiệm (experimental) kết hợp giữa trí tuệ nhân tạo (AI), đồ họa máy tính (3D) và tương tác người-máy (HCI) trên nền tảng Web.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ (stack) hiện đại, tối ưu cho hiệu suất thời gian thực:

*   **Ngôn ngữ & Công cụ:** **TypeScript** (đảm bảo type-safe cho các phép toán vector phức tạp) và **Vite** (bundler tốc độ cao). Sử dụng **Bun** làm môi trường thực thi và quản lý package.
*   **Đồ họa 3D:** **Three.js** là thư viện chủ đạo để dựng scene, quản lý ánh sáng và vật liệu.
*   **Trí tuệ nhân tạo (Computer Vision):**
    *   **MediaPipe Tasks Vision:** Dùng để theo dõi 21 điểm mốc (landmarks) của bàn tay trong không gian 3D với độ trễ cực thấp.
    *   **inferencejs & YOLOv8n (Nano):** Dùng cho module Nhận dạng hình ảnh (Visual Recognition), chạy mô hình object detection ngay trên trình duyệt thông qua WebGL GPU acceleration.
*   **Vật lý & Animation:**
    *   **Rapier3D:** Engine vật lý hiệu năng cao (viết bằng Rust/WASM) cho các giả lập va chạm phức tạp (Magnetic Clutter).
    *   **GSAP (GreenSock):** Xử lý các hiệu ứng chuyển cảnh và interpolation mượt mà.
    *   **Position Based Dynamics (PBD):** Kỹ thuật tự tùy biến để mô phỏng dây bóng đèn (Light Bulb) đảm bảo tính ổn định của hệ thống vật lý.
*   **Hậu kỳ (Post-processing):** Thư viện `postprocessing` để tạo các hiệu ứng Bloom (glowing), Chromatic Aberration, và LUT color grading (tạo cảm giác điện ảnh).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án tuân thủ mô hình **Orchestrator-Controller-Renderer**:

*   **Tính Module hóa tuyệt đối:** Mỗi thí nghiệm (Experiment) là một module độc lập nằm trong thư mục riêng (ví dụ: `src/cosmic-slash/`). Mỗi module có Controller riêng để quản lý logic và Renderer riêng cho phần hiển thị.
*   **Trung tâm điều phối (App.ts):** Đóng vai trò là "Main Brain", chịu trách nhiệm khởi tạo `HandTracker` (dùng chung cho tất cả các mode để tiết kiệm tài nguyên), quản lý vòng đời (lifecycle) của các module và điều hướng người dùng.
*   **Tách biệt tầng xử lý dữ liệu:** Hand Tracking không can thiệp trực tiếp vào 3D. Dữ liệu từ camera đi qua `HandTracker` -> được chuẩn hóa tọa độ -> đưa vào `GestureDetector` để định nghĩa các hành động "pinch" (bấm ngón tay), "fist" (nắm đấm) -> cuối cùng mới truyền tín hiệu đến các Controller.
*   **Thread Isolation (Cô lập luồng):** Đối với các tác vụ nặng như AI Inference, hệ thống sử dụng **Web Workers** để chạy các thuật toán nhận diện trong luồng riêng, giúp luồng hiển thị (Main UI thread) luôn duy trì ở mức 60 FPS mà không bị khựng (stutter).

---

### 3. Các kỹ thuật chính (Key Technical Techniques)

*   **Exponential Moving Average (EMA) Smoothing:** Tín hiệu từ camera thường bị rung (jitter). Hệ thống sử dụng thuật toán làm mượt EMA để các vật thể 3D di chuyển theo tay một cách tự nhiên, không bị giật.
*   **Inverse Quaternions for Local Space:** Trong module Voxel Builder, khi người dùng xoay khối hình, hệ thống sử dụng toán học Quaternion nghịch đảo để tính toán vị trí tay trong không gian nội bộ của khối, giúp người dùng có thể "vẽ" từ mọi góc độ.
*   **Object Pooling:** Trong Cosmic Slash, thay vì tạo/xóa liên tục các vật thể (gây rác bộ nhớ - GC spikes), hệ thống tái sử dụng các instance từ một "pool" có sẵn để tối ưu hiệu suất.
*   **Instanced Rendering:** Sử dụng `THREE.InstancedMesh` để vẽ hàng ngàn hạt (particles) hoặc khối voxel chỉ với một lệnh vẽ (draw call) duy nhất, giảm tải tối đa cho GPU.
*   **Hysteresis in Gesture Detection:** Sử dụng hai ngưỡng (threshold) khác nhau để bắt đầu và kết thúc một cử chỉ. Điều này ngăn chặn tình trạng "flickering" (tín hiệu bật tắt liên tục khi tay ở sát ngưỡng nhận diện).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng dữ liệu của Gesture Lab là một đường ống (pipeline) khép kín:

1.  **Input:** Trình duyệt truy cập Webcam thông qua MediaDevices API.
2.  **Detection:** MediaPipe phân tích từng frame hình ảnh, trích xuất 21 điểm tọa độ bàn tay theo dạng chuẩn hóa (0.0 đến 1.0).
3.  **Mapping:** Hệ thống ánh xạ (map) tọa độ 2D/3D từ camera sang không gian thế giới (World Space) của Three.js, có tính đến tỉ lệ màn hình (Aspect Ratio) và độ phân giải thực tế.
4.  **Gesture Logic:** `GestureDetector` phân tích khoảng cách giữa các đầu ngón tay để xác định ý định người dùng (ví dụ: ngón cái chạm ngón trỏ = Pinch).
5.  **Simulation & Logic:**
    *   Nếu là Mode vật lý: Cử chỉ của người dùng tạo ra các lực (force) tác động vào các rigid body trong Rapier3D.
    *   Nếu là Mode đồ họa: Cử chỉ làm thay đổi các giá trị Uniform trong Shaders (GLSL).
6.  **Rendering:** Three.js render scene 3D với các lớp post-processing.
7.  **Output:** Hiển thị kết quả lên Canvas 2D/3D trên trình duyệt người dùng.

### Kết luận
**Gesture Lab** không chỉ là một ứng dụng vui nhộn mà còn là một ví dụ mẫu về việc xử lý **Edge Computing** (tính toán tại biên - ngay trên trình duyệt). Việc thực hiện toàn bộ quy trình từ tracking đến rendering mà không cần server-side xử lý cho thấy sức mạnh của WebAssembly và WebGL trong kỷ nguyên web hiện đại.