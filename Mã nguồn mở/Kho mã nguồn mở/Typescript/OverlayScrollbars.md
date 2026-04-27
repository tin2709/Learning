Dựa trên mã nguồn và tài liệu của dự án **OverlayScrollbars (v2.x)**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của thư viện này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **TypeScript (Strict Mode):** Toàn bộ logic lõi được viết bằng TypeScript, giúp kiểm soát kiểu dữ liệu chặt chẽ, đặc biệt là các cấu hình Options và State phức tạp.
*   **Modern Browser APIs:**
    *   **ResizeObserver:** Theo dõi sự thay đổi kích thước của phần tử (thay vì dùng `iframe` hoặc `object` như v1).
    *   **MutationObserver:** Theo dõi sự thay đổi của DOM (thuộc tính, nội dung, node con) để cập nhật scrollbar tự động.
    *   **IntersectionObserver:** Sử dụng trong `TrinsicObserver` để phát hiện sự thay đổi về kích thước nội tại (intrinsic sizing).
    *   **ScrollTimeline API (Experimental):** Sử dụng các tính năng mới nhất để thực hiện hoạt ảnh scroll mượt mà và tối ưu hiệu suất (trong các trình duyệt hỗ trợ).
*   **CSS Custom Properties (Variables):** Sử dụng biến CSS để điều khiển vị trí, kích thước handle (`--os-viewport-percent`, `--os-scroll-percent`). Điều này giúp chuyển dịch gánh nặng tính toán từ JavaScript sang GPU của trình duyệt.
*   **SCSS:** Quản lý style có cấu trúc, module hóa cao.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OverlayScrollbars v2 chuyển từ hướng đối tượng (OOP) sang **Functional & Modular Architecture**:

*   **Framework Agnostic Core:** Logic lõi hoàn toàn độc lập với các framework UI. Nó giao tiếp với DOM thông qua các hàm wrapper trong thư mục `support/dom`.
*   **Plugin-driven Design:** Thư viện lõi cực kỳ nhỏ gọn. Các tính năng như: `ClickScroll` (click vào track để scroll), `ScrollbarsHiding` (kỹ thuật ẩn thanh cuộn gốc), `SizeObserver` (polyfill cho trình duyệt cũ) đều được tách thành các Plugin. Nếu không dùng đến, chúng sẽ bị **Treeshaking** để giảm bundle size.
*   **Segmented Setup:** Logic khởi tạo được chia thành 3 phần chính (`Setups`):
    1.  **ObserversSetup:** Thiết lập các "tai mắt" để lắng nghe thay đổi.
    2.  **StructureSetup:** Xây dựng cấu trúc DOM cần thiết (Host, Viewport, Padding, Content).
    3.  **ScrollbarsSetup:** Quản lý việc hiển thị, style và sự kiện của các thanh cuộn tùy chỉnh.
*   **State-driven UI:** Trạng thái của instance (`State`) được cập nhật thông qua luồng dữ liệu một chiều. Khi có thay đổi từ môi trường hoặc options, hệ thống sẽ tính toán lại các "Hints" (gợi ý thay đổi) và chỉ cập nhật những gì cần thiết.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Tối ưu hóa Hiệu suất (Reflow/Repaint):**
    *   **`createCache` utility:** Đây là kỹ thuật quan trọng nhất. Thư viện lưu lại giá trị cũ và chỉ thực hiện các thao tác DOM đắt đỏ (như `setStyles`) khi giá trị mới thực sự khác giá trị cũ.
    *   **Debouncing & Throttling:** Các thay đổi từ `ResizeObserver` hoặc `MutationObserver` được gom nhóm và xử lý có độ trễ (mặc định 33ms) để tránh làm treo trình duyệt khi DOM thay đổi liên tục.
*   **Kỹ thuật Ẩn thanh cuộn (Scrollbar Hiding):**
    *   Sử dụng kết hợp `margin` âm, `padding` bù trừ và thuộc tính `scrollbar-width: none` hoặc `::-webkit-scrollbar { display: none }` để ẩn thanh cuộn hệ thống mà vẫn giữ được khả năng scroll tự nhiên của trình duyệt.
*   **Xử lý hướng cuộn phi tiêu chuẩn (Non-default flow):** Thư viện có logic thông minh để xử lý `direction: rtl`, `flex-direction: column-reverse`, `writing-mode`. Nó thực hiện đo đạc thực tế (`getMeasuredScrollCoordinates`) để biết tọa độ bắt đầu và kết thúc của mỗi trình duyệt (vốn không đồng nhất).
*   **Event Hub:** Một hệ thống quản lý sự kiện nội bộ (`createEventListenerHub`) cho phép đăng ký và kích hoạt các sự kiện (`initialized`, `updated`, `scroll`, `destroyed`) một cách hiệu quả.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi tạo (Initialization):**
    *   Hệ thống nhận `Target` (phần tử cần áp dụng).
    *   Kiểm tra điều kiện hủy (ví dụ: trình duyệt di động có thanh cuộn chồng sẵn - overlaid).
    *   `StructureSetup` bao bọc nội dung gốc vào một cấu trúc DOM mới: `Host -> Padding -> Viewport -> Content`.
    *   Chuyển vị trí cuộn hiện tại của phần tử gốc sang `Viewport`.

2.  **Lắng nghe thay đổi (Observation):**
    *   Kích hoạt `SizeObserver` để theo dõi kích thước của Host.
    *   Kích hoạt `MutationObserver` để theo dõi sự thay đổi nội dung bên trong.
    *   Kích hoạt `TrinsicObserver` để theo dõi sự thay đổi từ `auto` height sang fixed height.

3.  **Vòng lặp Cập nhật (Update Loop):**
    *   Khi Observers phát hiện thay đổi -> Kích hoạt hàm `update`.
    *   **Bước 1:** `ObserversSetup` tính toán các `updateHints` (Ví dụ: `sizeChanged: true`).
    *   **Bước 2:** `StructureSetup` sử dụng các hints này để tính toán lại kích thước padding, overflow và scroll coordinates.
    *   **Bước 3:** `ScrollbarsSetup` nhận thông tin và cập nhật độ dài, vị trí của handle thông qua biến CSS.

4.  **Tương tác người dùng (Interaction):**
    *   Khi người dùng kéo handle hoặc click vào track: JavaScript tính toán tỷ lệ phần trăm cuộn dựa trên vị trí chuột và kích thước thực tế của viewport, sau đó gọi lệnh `scrollTo` gốc trên phần tử Viewport.
    *   Sự kiện `scroll` gốc của Viewport sẽ kích hoạt việc cập nhật lại vị trí handle (đồng bộ ngược lại).

5.  **Hủy (Destroy):**
    *   Xóa bỏ các Observers.
    *   Giải phóng các phần tử DOM đã tạo, đưa nội dung gốc trở lại vị trí cũ.
    *   Xóa bỏ các class và thuộc tính `data-os-*`.

### Tổng kết
OverlayScrollbars v2 là một ví dụ điển hình về việc xây dựng thư viện UI hiện đại: **Nhẹ hơn (với Treeshaking), Nhanh hơn (với Caching & CSS Variables) và Linh hoạt hơn (với Plugin system)** so với các giải pháp truyền thống.