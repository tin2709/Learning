Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Immich Power Tools** (một công cụ mở rộng không chính thức dành cho Immich):

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào hiệu suất và khả năng xử lý dữ liệu lớn:

*   **Framework chính:** **Next.js 14** (sử dụng Page Router). Đây là lựa chọn tối ưu để kết hợp giữa giao diện người dùng (Frontend) và các API trung gian (Backend) trong cùng một dự án.
*   **Quản lý dữ liệu & ORM:** **Drizzle ORM** kết hợp với trình điều khiển **pg (node-postgres)**. Dự án kết nối trực tiếp vào cơ sở dữ liệu PostgreSQL của Immich để thực hiện các truy vấn phức tạp mà API chính thức của Immich chưa hỗ trợ hoặc xử lý chậm.
*   **Giao diện (UI/UX):**
    *   **Tailwind CSS:** Xử lý phong cách dàn trang.
    *   **Shadcn UI (Radix UI):** Cung cấp các thành phần giao diện chất lượng cao như Dialog, Table, Popover.
    *   **TanStack Table (React Table):** Xử lý các bảng dữ liệu lớn (quản lý Album/People) với khả năng lọc và sắp xếp mạnh mẽ.
*   **Trí tuệ nhân tạo (AI):** **Google Gemini SDK**. Sử dụng mô hình Gemini 1.5/2.0 Flash để xử lý ngôn ngữ tự nhiên, chuyển đổi câu hỏi của người dùng thành các bộ lọc tìm kiếm dữ liệu.
*   **Xử lý hình ảnh & Video:**
    *   **Remotion:** Một kỹ thuật độc đáo để tạo video (tính năng Rewind) bằng code React.
    *   **Leaflet:** Thư viện bản đồ để hiển thị Heatmap vị trí ảnh.
    *   **React Window:** Kỹ thuật ảo hóa (Virtualization) để hiển thị lưới ảnh hàng nghìn mục mà không làm lag trình duyệt.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án đi theo mô hình **"Sidecar Tool"** (Công cụ hỗ trợ song song) với các tư duy kiến trúc đặc trưng:

*   **Hybrid Data Access (Truy cập dữ liệu hỗn hợp):** 
    *   Sử dụng **Immich API** cho các tác vụ thay đổi dữ liệu (Update, Delete, Create) để đảm bảo tính toàn vẹn.
    *   Sử dụng **Direct DB Access (SQL)** thông qua Drizzle cho các tác vụ đọc/phân tích (Analytics, Potential Albums) để đạt tốc độ tối đa.
*   **Proxy Pattern (Mẫu thiết kế ủy nhiệm):** Dự án triển khai một lớp `immich-proxy`. Vì API của Immich yêu cầu Token/API Key trong Header (dễ bị lộ nếu gọi trực tiếp từ trình duyệt), Power Tools đóng vai trò là một Proxy trung gian: Trình duyệt gọi Power Tools -> Power Tools thêm Header bảo mật -> Gọi Immich -> Trả kết quả.
*   **Stateless Share Links:** Tính năng chia sẻ ảnh được thiết kế dựa trên **JWT (JSON Web Token)**. Mọi bộ lọc (người, album, thời gian) được mã hóa vào Token, giúp tạo ra các link chia sẻ "không trạng thái", không cần lưu trữ thêm vào DB của Power Tools.
*   **Environment-Driven Configuration:** Toàn bộ cấu hình từ kết nối DB đến các khóa API (Gemini, Immich) đều được quản lý qua biến môi trường (`.env`), phù hợp hoàn hảo cho việc triển khai qua Docker.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Smart Merge (Gộp người thông minh):** Sử dụng toán tử **Cosine Distance** trên các Vector Embedding (thông qua `pgvector` trong Postgres) để tìm kiếm các khuôn mặt tương đồng với độ chính xác cao dựa trên ngưỡng (threshold) tùy chỉnh.
*   **Virtualized Asset Grid:** Triển khai kỹ thuật **Windowing** (chỉ render những tấm ảnh đang hiển thị trong khung nhìn). Điều này cực kỳ quan trọng với các thư viện ảnh cá nhân có thể lên tới hàng chục nghìn file.
*   **Natural Language Parser:** Kỹ thuật **Prompt Engineering** gửi câu lệnh người dùng tới Gemini kèm theo Schema định sẵn để nhận về JSON cấu trúc (ví dụ: "ảnh biển năm ngoái" -> `{city: "beach", takenAfter: "2023-01-01"}`).
*   **Thumbnail Proxying:** Do Immich không cho phép truy cập trực tiếp đường dẫn ảnh nếu không có quyền, dự án xây dựng một API trả về luồng dữ liệu nhị phân (Binary Stream) cho ảnh thumbnail và video, giúp hiển thị mượt mà trên UI.
*   **Bulk Date Offsetting:** Kỹ thuật xử lý thời gian hàng loạt, cho phép cộng/trừ năm, ngày, giờ vào metadata của hàng loạt ảnh để sửa lỗi sai múi giờ khi upload.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Giai đoạn Khởi tạo:** Khi khởi chạy qua Docker, công cụ kết nối song song vào **Immich API** (qua URL) và **Immich Postgres DB**.
2.  **Luồng Xác thực:** 
    *   Người dùng đăng nhập bằng Email/Password của Immich.
    *   Power Tools gọi API của Immich để lấy Access Token.
    *   Token này được lưu vào **Http-only Cookie** để bảo mật.
3.  **Luồng Xử lý "Missing Location" (Vị trí bị thiếu):**
    *   Dân dùng SQL truy vấn DB: Tìm các Asset có `latitude` là NULL nhưng có dữ liệu EXIF hợp lệ.
    *   Hiển thị danh sách theo ngày/Album.
    *   Người dùng chọn ảnh và chọn vị trí trên bản đồ Leaflet.
    *   Power Tools gọi API `PUT /assets` của Immich để cập nhật tọa độ thực tế.
4.  **Luồng Tìm kiếm thông minh (Find Assets):**
    *   Người dùng nhập câu hỏi -> Gửi tới Gemini API.
    *   Gemini trả về các tham số lọc (Ví dụ: `personIds`, `city`).
    *   Power Tools kết hợp các tham số này để gọi API `Smart Search` của chính Immich hoặc truy vấn trực tiếp DB.
    *   Kết quả được hiển thị qua lưới ảnh ảo hóa.
5.  **Luồng Rewind (Tổng kết năm):**
    *   Hệ thống đếm số lượng ảnh, các quốc gia đã đi, những người xuất hiện nhiều nhất trong năm qua SQL `GROUP BY`.
    *   Dữ liệu này được đẩy vào **Remotion Player**.
    *   React render các "Scene" (Cảnh quay) động theo thời gian thực để người dùng xem như một video ngắn.

### Tổng kết
**Immich Power Tools** là một ví dụ điển hình về việc xây dựng công cụ vệ tinh (Satellite App). Nó không thay thế Immich mà bổ sung những tính năng "nặng" về xử lý dữ liệu hàng loạt và phân tích thông minh, tận dụng tối đa sức mạnh của SQL trực tiếp và AI hiện đại.