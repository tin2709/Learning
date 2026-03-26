Dựa trên mã nguồn và tài liệu của dự án **Kokraf**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật và kiến trúc của ứng dụng 3D Modeling này:

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Three.js (Rendering Engine):** Sử dụng làm nền tảng hiển thị đồ họa 3D. Tuy nhiên, Three.js chỉ đóng vai trò là "lớp hiển thị", không phải là lớp dữ liệu gốc.
*   **VEF (Vertex-Edge-Face) Mesh Representation:** Đây là công nghệ quan trọng nhất. Kokraf tự xây dựng một cấu trúc dữ liệu lưới (Adjacency Mesh) tùy chỉnh. Thay vì dùng `BufferGeometry` của Three.js (vốn tối ưu cho GPU nhưng khó chỉnh sửa cấu trúc), Kokraf dùng VEF để quản lý các mối quan hệ láng giềng giữa đỉnh, cạnh và mặt, cho phép thực hiện các thao tác phức tạp như Extrude, Bevel, Loop Cut.
*   **Vanilla JavaScript & ESM:** Ứng dụng không sử dụng framework (như React/Vue). Nó dựa vào native ES Modules và `importmap` để quản lý dependencies, giúp giảm overhead và tối ưu hiệu suất cho các tác vụ tính toán hình học nặng.
*   **Supabase & Edge Functions:** Backend được xử lý bởi Supabase (PostgreSQL + Auth). Các logic về thanh toán (Paddle webhook) và quản lý Credit/Subscription được triển khai qua Deno Edge Functions.
*   **IndexedDB (Local Storage):** Sử dụng `Storage.js` để lưu trạng thái scene hiện tại xuống trình duyệt, giúp người dùng không bị mất dữ liệu khi refresh trang.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Decoupling Logic & View (Tách biệt logic và hiển thị):** 
    *   **Logic:** Mọi thao tác mô hình hóa diễn ra trên đối tượng `MeshData`. 
    *   **View:** Sau khi `MeshData` thay đổi, hệ thống sử dụng `MeshRendererAdapter` để chuyển đổi cấu trúc VEF thành `BufferGeometry` cho Three.js render. 
    *   *Lợi ích:* Cho phép hỗ trợ các đa giác (ngon/quads) trong khi Three.js chỉ hiểu tam giác (triangles).
*   **Command Pattern (Mẫu lệnh):** Thư mục `js/commands/` chứa hàng loạt các lớp (Add, Remove, SetPosition, Extrude...). Mỗi hành động của người dùng được đóng gói thành một đối tượng `Command`. Điều này cho phép hệ thống triển khai tính năng **Undo/Redo** một cách nhất quán và mạnh mẽ.
*   **Signal-based Architecture:** Sử dụng `Signals.js` (mô hình Observer) để các thành phần giao tiếp với nhau mà không bị phụ thuộc trực tiếp (loose coupling). Ví dụ: Khi một đối tượng được chọn, một tín hiệu được phát đi, `Sidebar` sẽ tự cập nhật thuộc tính mà không cần `Selection` phải biết `Sidebar` là ai.
*   **Manager/Controller Pattern:** Các lớp `SceneManager`, `CameraManager`, `ControlsManager` đóng vai trò quản lý vòng đời của từng module chuyên biệt trong Editor.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Topology Manipulation:** Trong `js/vertex/`, Kokraf xử lý các bài toán hình học topo như:
    *   *Dissolve:* Xóa đỉnh/cạnh nhưng vẫn giữ nguyên bề mặt.
    *   *Subdivide:* Chia nhỏ các mặt để tăng độ chi tiết.
    *   *Merge:* Gộp các đỉnh gần nhau.
*   **Custom Orbit Controls:** `QuaternionOrbitControls.js` là bản thực thi tùy chỉnh sử dụng Quaternion (số ảo bốn chiều) để xoay camera, tránh hiện tượng *Gimbal Lock* (khóa trục) mà các hệ thống dùng góc Euler thường gặp.
*   **Triangulation (Tam giác hóa):** Sử dụng thư viện `earcut` để tính toán cách chia các mặt đa giác phức tạp (có thể bị lõm) thành các tam giác để Three.js có thể hiển thị chính xác.
*   **Atomic Actions:** Các hành động trong `EditActions.js` và `ObjectActions.js` luôn đảm bảo tính toàn vẹn của dữ liệu lưới trước và sau khi thực hiện lệnh.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một thao tác người dùng (ví dụ: Di chuyển một đỉnh):

1.  **Input (Đầu vào):** Người dùng tương tác qua `TransformControls` (gizmo) hoặc phím tắt.
2.  **Action (Hành động):** `EditActions` nhận sự kiện, lấy danh sách các đỉnh đang chọn từ `EditSelection`.
3.  **Command Execution (Thực thi lệnh):** Một `SetVertexPositionCommand` được tạo ra. Nó lưu lại vị trí cũ (`beforeMeshData`).
4.  **Data Update (Cập nhật dữ liệu):** `VertexEditor` thay đổi tọa độ đỉnh trong `MeshData`.
5.  **Mesh Sync (Đồng bộ):** `MeshRendererAdapter` được gọi để "build" lại `BufferGeometry` mới dựa trên `MeshData` vừa cập nhật.
6.  **Rendering (Hiển thị):** `Renderer.js` xóa frame cũ và vẽ lại frame mới với geometry đã thay đổi.
7.  **History & Persistence:** Lệnh được đẩy vào stack của `History` (để Undo) và dữ liệu được tự động lưu vào IndexedDB qua `Storage.js`.

### Tổng kết
Kokraf là một dự án có độ khó kỹ thuật cao, tập trung vào việc tái định nghĩa cách quản lý dữ liệu 3D trên trình duyệt. Thay vì dựa dẫm hoàn toàn vào các tính năng có sẵn của Three.js, nó xây dựng một **Geometric Kernel** riêng (VEF Mesh), biến trình duyệt thành một công cụ CAD/DCC thực thụ tương tự như phiên bản web thu nhỏ của Blender.