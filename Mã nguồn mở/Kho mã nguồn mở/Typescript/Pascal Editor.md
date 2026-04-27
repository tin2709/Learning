Dựa trên tài liệu và cấu trúc mã nguồn của Pascal Editor, dưới đây là phân tích chuyên sâu về hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology)

Pascal Editor được xây dựng trên một ngăn xếp công nghệ hiện đại nhất (Modern Tech Stack) cho đồ họa Web:

*   **Đồ họa & Rendering:**
    *   **Three.js (WebGPU Renderer):** Sử dụng API đồ họa thế hệ mới giúp tối ưu hóa hiệu suất tính toán song song trên GPU, vượt qua giới hạn của WebGL.
    *   **React Three Fiber (R3F):** Thư viện "bridge" đưa Three.js vào hệ sinh thái React, cho phép quản lý vòng đời các đối tượng 3D như các React Component.
    *   **Three-bvh-csg:** Công nghệ then chốt để thực hiện các phép toán Boolean (cắt, hợp, giao) trên hình học 3D trong thời gian thực (ví dụ: đục lỗ cửa sổ trên tường).
*   **Quản lý trạng thái (State Management):**
    *   **Zustand + Zundo:** Zustand cung cấp store nhẹ, hiệu năng cao để quản lý hàng ngàn node trong scene. Zundo hỗ trợ tính năng Undo/Redo mạnh mẽ với cơ chế quay ngược trạng thái.
*   **Xử lý dữ liệu & Định kiểu:**
    *   **Zod Schema:** Toàn bộ cấu trúc dữ liệu của tòa nhà (Wall, Slab, Item...) được định nghĩa bằng Zod, đảm bảo dữ liệu luôn đúng định dạng khi lưu trữ hoặc truyền tải.
*   **Cơ sở hạ tầng (Infrastructure):**
    *   **Turborepo & Bun:** Quản lý monorepo và gói thư viện với tốc độ cực nhanh.
    *   **Next.js 16 & React 19:** Tận dụng các tính năng mới nhất như Server Actions và cải tiến hiệu suất render.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống áp dụng mô hình phân rã trách nhiệm (Separation of Concerns) rất rõ ràng thông qua 3 lớp:

*   **Lớp Dữ liệu (Package `core`):** Đóng vai trò là "Single Source of Truth". Nó chứa schema, logic tính toán hình học (Systems) và quản lý trạng thái thô. Lớp này hoàn toàn không phụ thuộc vào UI hay môi trường render 3D cụ thể nào (Headless).
*   **Lớp Hiển thị (Package `viewer`):** Chịu trách nhiệm render dữ liệu từ `core`. Nó cung cấp các "sensible defaults" (camera, ánh sáng, post-processing) và quản lý việc hiển thị các node thông qua các Renderers.
*   **Lớp Ứng dụng (App `editor`):** Là lớp tích hợp, cung cấp bộ công cụ tương tác (Tools), giao diện người dùng (UI) và các logic đặc thù cho việc biên tập.
*   **Cấu trúc Node Phẳng (Flat Hierarchy):** Thay vì lưu trữ dạng cây lồng nhau (nested tree), Pascal lưu trữ node trong một dictionary phẳng (`Record<id, Node>`). Điều này giúp việc cập nhật node ở bất kỳ cấp nào cũng đạt tốc độ $O(1)$ và cực kỳ dễ dàng để đồng bộ hóa xuống IndexedDB.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Mô hình "Dirty Nodes" & Systems:** Đây là kỹ thuật tối ưu hóa hiệu năng cực kỳ hiệu quả. Khi một thuộc tính (như độ dày tường) thay đổi, ID của node đó được đưa vào một `Set` gọi là `dirtyNodes`. Các System (chạy trong vòng lặp `useFrame`) sẽ chỉ tính toán lại hình học cho những node nằm trong danh sách này, thay vì tính toán lại toàn bộ scene.
*   **Scene Registry:** Một giải pháp "lookup" ngược. Thông thường, React quản lý dữ liệu đi xuống, nhưng các System cần truy cập trực tiếp vào đối tượng Three.js để thay đổi mesh. `useRegistry` cho phép đăng ký tham chiếu (ref) của đối tượng 3D vào một Map toàn cục để các System truy vấn nhanh qua ID.
*   **Spatial Grid Manager:** Sử dụng lưới không gian 2D để quản lý va chạm và vị trí. Thay vì duyệt qua hàng ngàn đối tượng để kiểm tra chồng lấn, hệ thống chỉ kiểm tra các ô lưới xung quanh, giúp tính năng đặt đồ đạc (Item placement) diễn ra mượt mà.
*   **Dependency Injection qua React Children:** Viewer không import bất kỳ thứ gì từ Editor. Editor sẽ "tiêm" các công cụ và hệ thống tương tác vào Viewer thông qua props hoặc children, giữ cho Viewer luôn độc lập và có thể tái sử dụng (ví dụ: làm trang xem trước 3D tĩnh).

### 4. Luồng hoạt động hệ thống (System Operational Flow)

Luồng xử lý một hành động người dùng diễn ra như sau:

1.  **Input (Tương tác):** Người dùng kéo một bức tường bằng `WallTool`.
2.  **Mutation (Thay đổi trạng thái):** Tool gọi hàm `updateNode()` của `useScene` (Zustand).
3.  **Flagging (Đánh dấu):** Store cập nhật dữ liệu node và tự động thêm ID của tường đó vào `dirtyNodes`.
4.  **Geometry Processing (Xử lý hình học):**
    *   `WallSystem` nhận diện node bị "bẩn" trong frame tiếp theo.
    *   Nó lấy mesh từ `sceneRegistry`.
    *   Tính toán lại tọa độ các đỉnh (vertices), xử lý vát góc (mitering) và cắt lỗ thủng (CSG).
5.  **Reactive Rendering (Render lại):** R3F nhận thấy mesh đã thay đổi và gửi lệnh render mới tới GPU thông qua WebGPU API.
6.  **Persistence (Lưu trữ):** Middleware lưu trạng thái mới vào IndexedDB và lưu snapshot vào Zundo để hỗ trợ Undo.

### Tổng kết
Kiến trúc của Pascal Editor không chỉ là một ứng dụng React thông thường, mà là một **"Geometry Engine"** được bao bọc bởi React. Sự kết hợp giữa **Flat State**, **Dirty Flags** và **Scene Registry** tạo ra một hệ thống có khả năng mở rộng (scalability) cao, đủ sức xử lý các công trình kiến trúc phức tạp trên trình duyệt.