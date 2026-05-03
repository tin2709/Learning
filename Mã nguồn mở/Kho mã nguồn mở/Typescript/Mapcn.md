Dựa trên mã nguồn của dự án **mapcn**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

### 1. Công nghệ cốt lõi (Core Stack)
Dự án được xây dựng dựa trên sự kết hợp của ba trụ cột chính:
*   **MapLibre GL (Engine bản đồ):** Đây là "trái tim" của hệ thống. Thay vì sử dụng Google Maps hay Mapbox (có phí), dự án chọn MapLibre GL - một nhánh mã nguồn mở mạnh mẽ, sử dụng WebGL để render bản đồ vector với hiệu suất cực cao.
*   **Tailwind CSS (Styling):** Sử dụng Tailwind CSS v4 để tùy chỉnh giao diện các thành phần UI bao quanh bản đồ (như cards, sidebars, tooltips). Điều này giúp mapcn có giao diện rất hiện đại và đồng bộ với hệ sinh thái **shadcn/ui**.
*   **Next.js & React 19:** Framework để xây dựng ứng dụng web hiện đại, tận dụng Server Components và cấu trúc thư mục App Router.
*   **Shadcn Registry System:** Đây là điểm đặc biệt nhất. Thay vì cài đặt như một thư viện NPM thông thường (`node_modules`), dự án sử dụng hệ thống **Registry**. Người dùng "sở hữu" code bằng cách copy-paste hoặc dùng CLI (`npx shadcn add`) để tải component trực tiếp vào thư mục dự án của mình.

### 2. Tư duy Kiến trúc (Architectural Mindset)
Kiến trúc của mapcn tuân thủ nguyên lý **"Compound Components"** (Thành phần phức hợp):

*   **Tính đóng gói (Encapsulation):** Thành phần `<Map />` đóng vai trò là container gốc, quản lý việc khởi tạo bản đồ (initialization) và cung cấp `Context` cho các thành phần con.
*   **React Context API:** Sử dụng một `MapContext` để chia sẻ thực thể (instance) của bản đồ MapLibre cho các thành phần con như `<MapMarker />`, `<MapRoute />`, `<MapControls />`. Điều này giúp code trông rất sạch và có tính khai báo (declarative).
*   **Theme-Aware (Nhận diện giao diện):** Hệ thống có tư duy xử lý theme (Light/Dark) tự động. Khi ứng dụng đổi màu nền, bản đồ cũng tự động thay đổi `style URL` của tile provider (mặc định là CARTO) để phù hợp với giao diện người dùng.
*   **Hiệu suất (Performance):** Dự án phân tách rõ hai cách tiếp cận dữ liệu:
    *   **DOM Markers:** Dành cho số lượng điểm ít (dưới vài trăm), sử dụng React để render UI phức tạp bên trong Marker.
    *   **Layer-based (GeoJSON):** Dành cho dữ liệu lớn (như bản đồ nhiệt - Heatmap hoặc Cluster), render trực tiếp bằng WebGL thông qua các layer của MapLibre để đảm bảo không bị lag.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Custom Hooks (`useMap`):** Một hook quan trọng giúp các component con truy cập vào đối tượng `map` một cách an toàn. Nó kiểm tra xem bản đồ đã load xong chưa trước khi thực hiện các thao tác như thêm marker hay vẽ đường đi.
*   **Ref Forwarding:** Sử dụng `forwardRef` để cho phép lập trình viên truy cập trực tiếp vào thực thể MapLibre gốc từ bên ngoài. Điều này cực kỳ quan trọng cho các thao tác phức tạp như `flyTo()`, `easeTo()` mà React thuần túy khó quản lý.
*   **Đồng bộ hóa Viewport (Controlled vs Uncontrolled):** mapcn hỗ trợ cả chế độ tự quản lý vị trí (uncontrolled) và chế độ bị điều khiển bởi state bên ngoài (controlled) thông qua prop `viewport` và `onViewportChange`.
*   **Xử lý hình học (Geometry Handling):** 
    *   **MapArc:** Sử dụng kỹ thuật tính toán đường cong Bézier bậc 2 (Quadratic Bézier) để vẽ các đường cung nối giữa các thành phố, tạo hiệu ứng thị giác chuyên nghiệp cho bản đồ logistics.
    *   **Route Rendering:** Tích hợp với OSRM (Open Source Routing Machine) để lấy dữ liệu đường đi thực tế từ tọa độ và vẽ chúng lên bản đồ.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Khởi tạo (Mounting):** 
    *   Component `<Map />` được render. 
    *   `useEffect` khởi tạo `new maplibregl.Map()`.
    *   Khi bản đồ sẵn sàng (sự kiện `load`), `isLoaded` chuyển thành `true` và `Context` được cập nhật.
2.  **Giai đoạn Đăng ký thành phần (Component Registration):**
    *   Các component con như `<MapMarker />` hoặc `<MapRoute />` được gắn vào.
    *   Chúng sử dụng `map instance` từ Context để gọi các hàm MapLibre tương ứng như `addSource()` và `addLayer()`.
3.  **Giai đoạn Tương tác (Interaction):**
    *   Người dùng tương tác (click, hover). Hệ thống bắt các sự kiện từ MapLibre và chuyển đổi ngược lại thành các React props (`onClick`, `onHover`).
    *   Popup/Tooltip được hiển thị bằng cách đồng bộ tọa độ màn hình (pixel) với tọa độ địa lý (lng/lat).
4.  **Giai đoạn Giải phóng (Unmounting):**
    *   Khi một component (ví dụ một Marker) bị xóa khỏi React tree, nó sẽ gọi hàm cleanup (`marker.remove()` hoặc `map.removeLayer()`) để tránh rò rỉ bộ nhớ (memory leak).

### Tóm lại
**mapcn** không chỉ là một wrapper cho MapLibre, mà là một **Design System cho Bản đồ**. Nó chuyển đổi tư duy lập trình bản đồ từ "thao tác trực tiếp trên canvas" sang "lập trình khai báo bằng React component", giúp việc xây dựng các dashboard có bản đồ trở nên dễ dàng như việc xây dựng một form hay một table thông thường.