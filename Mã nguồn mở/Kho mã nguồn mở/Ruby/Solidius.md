Dựa trên các tài liệu và cấu trúc mã nguồn của dự án **Solidus**, dưới đây là phân tích chi tiết về công nghệ cốt lõi, tư duy kiến trúc, kỹ thuật lập trình và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)
Solidus là một nền tảng Thương mại điện tử (E-commerce) mã nguồn mở mạnh mẽ, được xây dựng trên hệ sinh thái của Ruby.

*   **Ngôn ngữ & Framework chính:** Ruby (chiếm >83%) và **Ruby on Rails**. Hệ thống sử dụng kiến trúc Rails Engine để đóng gói các tính năng.
*   **Frontend (Admin mới):** 
    *   **Tailwind CSS:** Sử dụng cho giao diện quản trị hiện đại (`solidus_admin`).
    *   **Hotwire (Turbo & StimulusJS):** Thay thế các framework SPA phức tạp bằng cách gửi HTML qua dây (HTML over the wire), giúp tăng tốc độ phản hồi mà không cần viết quá nhiều JavaScript.
    *   **ViewComponent (của GitHub):** Dùng để xây dựng các thành phần giao diện (UI components) có tính bao đóng, dễ kiểm thử và tái sử dụng.
*   **Cơ sở dữ liệu:** Hỗ trợ linh hoạt thông qua ActiveRecord (PostgreSQL, MySQL, SQLite).
*   **Quản lý tài nguyên tĩnh:** Importmap và Sprockets/Vite (tùy cấu hình) để xử lý JS/CSS.
*   **Kiểm thử:** Sử dụng **RSpec** (cho Ruby), **FactoryBot** (tạo dữ liệu mẫu) và **Capybara** (kiểm thử tích hợp giao diện).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Solidus kế thừa tư duy từ Spree nhưng tập trung vào sự ổn định và khả năng tùy biến cao cho các doanh nghiệp lớn.

*   **Kiến trúc Modular Monolith (Monorepo):** Toàn bộ dự án được chia thành nhiều "Gems" nhỏ trong cùng một kho chứa:
    *   `solidus_core`: Chứa logic nghiệp vụ cốt lõi (Model, Migration, Service).
    *   `solidus_api`: Cung cấp RESTful API.
    *   `solidus_backend`: Giao diện Admin cũ (legacy).
    *   `solidus_admin`: Giao diện Admin mới sử dụng công nghệ hiện đại.
    *   `solidus_promotions`: Hệ thống khuyến mãi mới.
*   **Tách biệt logic nghiệp vụ:** Solidus cố gắng giữ `core` chỉ chứa logic thương mại (Order, Inventory, Tax, Payment). Các giao diện (Frontend/API) chỉ là các lớp bao bọc bên ngoài.
*   **Khả năng mở rộng (Extensibility):** Thay vì sửa trực tiếp vào mã nguồn (core), Solidus khuyến khích sử dụng **Extensions** và kỹ thuật **Decorators** (ghi đè hoặc mở rộng class của Rails) để tùy biến tính năng.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Máy trạng thái (State Machines):** Sử dụng rộng rãi cho `Order`, `Payment`, `Shipment` và `InventoryUnit`. Kỹ thuật này giúp quản lý các luồng phức tạp (ví dụ: một đơn hàng phải đi từ trạng thái `cart` -> `address` -> `delivery` -> `payment` -> `complete`).
*   **Calculators & Adjusters:** 
    *   **Calculators:** Các lớp chuyên biệt để tính toán thuế, phí vận chuyển, giảm giá (ví dụ: `FlatRate`, `PercentOnLineItem`).
    *   **Adjusters:** Hệ thống tự động cập nhật giá trị đơn hàng khi có thay đổi về sản phẩm hoặc khuyến mãi.
*   **Preference System:** Một hệ thống lưu trữ cấu hình linh hoạt cho phép lưu các cài đặt của store, phương thức thanh toán ngay trong database dưới dạng metadata hoặc cấu hình tĩnh.
*   **Component-Driven UI:** Trong `solidus_admin`, việc áp dụng `ViewComponent` giúp chia nhỏ giao diện thành các phần nhỏ (như `Badge`, `Button`, `Table`), mỗi phần có logic Ruby và template ERB riêng biệt.
*   **Blueprint Serialization:** Sử dụng `Blueprinter` để chuyển đổi các đối tượng Ruby thành JSON cho API, nhanh và sạch hơn so với Jbuilder truyền thống.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng đặt hàng (Checkout Flow)
Đây là luồng quan trọng nhất:
1.  **Cart:** Khách hàng thêm sản phẩm vào giỏ hàng (`Spree::Order`).
2.  **Processing:** Khi bắt đầu thanh toán, hệ thống sử dụng máy trạng thái để dẫn dắt qua các bước (Address, Delivery, Payment).
3.  **Validation:** Ở mỗi bước, hệ thống kiểm tra tồn kho (`InventoryUnit`), tính toán phí vận chuyển và thuế thông qua các `Calculators`.
4.  **Completion:** Khi thanh toán thành công, đơn hàng chuyển sang trạng thái `complete`, trừ tồn kho và gửi email xác nhận.

#### B. Luồng khuyến mãi (Promotion System)
1.  **Activators:** Hệ thống kiểm tra xem có sự kiện nào kích hoạt khuyến mãi không (ví dụ: thêm sản phẩm vào giỏ).
2.  **Rules:** Kiểm tra điều kiện (ví dụ: đơn hàng trên 500k, mã code là "SALE").
3.  **Actions:** Nếu thỏa mãn, áp dụng hành động (ví dụ: giảm giá 10% hoặc miễn phí vận chuyển) dưới dạng các `Adjustments` gắn vào đơn hàng.

#### C. Luồng quản trị hiện đại (Modern Admin Interaction)
1.  Người dùng thực hiện hành động (ví dụ: nhấn "Edit product").
2.  **Turbo** gửi request và chỉ cập nhật phần HTML cần thiết trong `turbo_frame_tag`.
3.  **StimulusJS** xử lý các tương tác nhỏ tại chỗ (như đóng mở modal, chọn ngày) mà không cần tải lại trang.
4.  Dữ liệu được lưu thông qua các `ResourcesController` chuẩn Rails nhưng được tối ưu để trả về các Component UI.

### Kết luận
Solidus không chỉ là một công cụ bán hàng mà là một **E-commerce Framework**. Nó cung cấp khung xương cực kỳ vững chắc về logic (Core) và máy trạng thái đơn hàng, đồng thời cho phép lập trình viên tự do xây dựng giao diện phía trước (Frontend) thông qua API hoặc các công nghệ hiện đại như Hotwire.