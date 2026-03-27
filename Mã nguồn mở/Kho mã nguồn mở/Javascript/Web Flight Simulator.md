Dựa trên cấu trúc mã nguồn và mô tả của dự án **Web Flight Simulator**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật:

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **CesiumJS:** Trái tim của hệ thống địa lý. Nó chịu trách nhiệm hiển thị quả địa cầu kỹ thuật số (Digital Twin Earth), xử lý dữ liệu địa hình 3D (Terrain) và hình ảnh vệ tinh thực tế theo thời gian thực từ các nguồn dữ liệu GIS.
*   **Three.js:** Công cụ đồ họa 3D dùng để hiển thị các đối tượng cục bộ có độ chi tiết cao như máy bay F-15, tên lửa, đạn, hiệu ứng hạt (lửa phản lực, khói, nổ) và ánh sáng.
*   **Vite:** Công cụ đóng gói (bundler) hiện đại, giúp tăng tốc độ phát triển (HMR) và tối ưu hóa hiệu suất khi chạy trên trình duyệt.
*   **Web Audio API (qua Three.js Audio):** Xử lý âm thanh không gian, thay đổi cao độ động dựa trên tốc độ động cơ và gió.
*   **ShaderMaterial (GLSL):** Sử dụng các chương trình shader tùy chỉnh để tạo hiệu ứng lửa phản lực (Jet Flame) chân thực mà không tốn nhiều tài nguyên như hệ thống hạt truyền thống.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án sử dụng **Kiến trúc Rendering Hỗn hợp (Hybrid Rendering Architecture)**:

*   **Phân tách không gian:** 
    *   *Cesium* quản lý hệ tọa độ toàn cầu (WGS84) và quy mô cực lớn của trái đất.
    *   *Three.js* quản lý hệ tọa độ cục bộ xung quanh máy bay để đảm bảo các phép tính vật lý và va chạm diễn ra mượt mà ở quy mô nhỏ mà không bị lỗi làm tròn số nổi (floating point error) khi ở xa gốc tọa độ.
*   **Đồng bộ hóa Camera (Camera Synchronization):** Hệ thống thực hiện đồng bộ vị trí và hướng của camera giữa Cesium và Three.js trong mỗi khung hình, tạo cảm giác hai engine đang chạy trên một không gian duy nhất.
*   **Thiết kế hướng Module:** Các tính năng được chia thành các hệ thống độc lập:
    *   `PlanePhysics`: Xử lý toán học về bay.
    *   `WeaponSystem`: Quản lý logic vũ khí và khóa mục tiêu.
    *   `NPCSystem`: Điều khiển AI máy bay địch.
    *   `HUD`: Lớp hiển thị UI (Heads-Up Display) tách biệt với logic game.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Chuyển đổi tọa độ (Coordinate Mapping):** Sử dụng các hàm toán học trong `src/utils/math.js` và API của Cesium (`Cartographic`, `Cartesian3`) để dịch chuyển vị trí máy bay từ kinh độ/vĩ độ sang không gian 3D và ngược lại.
*   **Vật lý Arcade (Arcade Physics):** Thay vì mô phỏng khí động học phức tạp (như Microsoft Flight Simulator), dự án sử dụng các kỹ thuật *Lerp (Linear Interpolation)* và *Quaternions* để tạo cảm giác bay mượt mà, dễ điều khiển kiểu trò chơi điện tử.
*   **Lock-on Logic:** Hệ thống vũ khí sử dụng *Dot Product (Tích vô hướng)* để kiểm tra xem máy bay địch có nằm trong tầm nhìn (FOV) của tên lửa hay không để thực hiện quá trình "khóa mục tiêu".
*   **Hiệu ứng hạt tối ưu:** Thay vì tạo hàng ngàn Object, hệ thống sử dụng một lớp `particles.js` dùng chung để quản lý vòng đời của các vụ nổ và mảnh vỡ, giúp duy trì FPS ổn định.
*   **Reverse Geocoding:** Sử dụng OpenStreetMap API để xác định tên khu vực (ví dụ: "Hanoi, Vietnam") dựa trên tọa độ GPS hiện tại của máy bay, tăng tính chân thực.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Khởi tạo (Initialization):**
    *   Định vị người dùng qua IP để xác định điểm xuất phát mặc định.
    *   Khởi tạo Cesium World (Globe) và Three.js Scene.
    *   Tải mô hình 3D F-15 (GLB) và các tài nguyên âm thanh.
2.  **Vòng lặp Game (The Main Loop):**
    *   **Input:** Nhận tín hiệu từ bàn phím (W/A/S/D, Arrows) và chuột.
    *   **Physics Update:** Tính toán vận tốc, hướng (Heading), độ dốc (Pitch), độ nghiêng (Roll).
    *   **Cesium Update:** Cập nhật vị trí camera toàn cầu để hiển thị địa hình tương ứng.
    *   **Three.js Update:** Di chuyển mô hình máy bay, cập nhật vị trí tên lửa, tính toán va chạm.
    *   **Sync:** Đồng bộ hóa hai Engine.
    *   **UI/HUD:** Cập nhật các chỉ số tốc độ, độ cao, la bàn trên màn hình.
3.  **Hệ thống Chiến đấu:**
    *   Khi người dùng bắn, một thực thể vũ khí mới được tạo ra trong Three.js.
    *   Hệ thống kiểm tra khoảng cách giữa đạn/tên lửa và NPC.
    *   Nếu trúng, kích hoạt hiệu ứng nổ và cộng điểm.
4.  **Xử lý va chạm địa hình:**
    *   Mỗi khung hình, hệ thống kiểm tra độ cao của máy bay so với độ cao mặt đất thực tế từ Cesium. Nếu `Alt <= TerrainHeight`, kích hoạt trạng thái "Crashed".

### Tổng kết
Dự án là một minh chứng xuất sắc cho việc kết hợp sức mạnh của **GIS (Cesium)** và **Đồ họa Game (Three.js)**. Nó giải quyết bài toán hiển thị cả thế giới nhưng vẫn giữ được chi tiết của một trò chơi hành động, điều mà ít engine 3D web nào làm được nếu chỉ chạy đơn độc.