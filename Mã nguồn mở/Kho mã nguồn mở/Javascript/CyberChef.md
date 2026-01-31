Dựa trên cấu trúc thư mục và nội dung các file cốt lõi của dự án **CyberChef**, dưới đây là phân tích chi tiết về kiến trúc và cách thức hoạt động của hệ thống này:

---

### 1. Công Nghệ Cốt Lõi (Core Technology)

CyberChef là một ứng dụng **Client-side** hoàn toàn, có nghĩa là mọi xử lý diễn ra trên trình duyệt của người dùng mà không cần máy chủ xử lý dữ liệu.

*   **Ngôn ngữ:** Sử dụng **Modern JavaScript (ES6+)** với hệ thống **ES Modules (.mjs)**. Điều này giúp mã nguồn tường minh, dễ quản lý và tận dụng được các tính năng mới nhất của JavaScript.
*   **Môi trường chạy:** Hỗ trợ cả **Trình duyệt (Browser)** và **Node.js**. Dự án có các file wrapper riêng (`src/node/`) để CyberChef có thể được sử dụng như một thư viện lập trình.
*   **Build Tools (Công cụ đóng gói):**
    *   **Webpack 5:** Dùng để đóng gói tài nguyên, xử lý các tệp CSS, hình ảnh và tối ưu hóa mã nguồn.
    *   **Grunt:** Đóng vai trò Task Runner để thực hiện các công việc như tạo file cấu hình (`generateConfig.mjs`), chạy test, và quản lý quy trình build.
    *   **Babel:** Chuyển đổi mã JavaScript mới về phiên bản tương thích với các trình duyệt cũ hơn.
*   **Thư viện bên thứ ba:** Tích hợp rất nhiều thư viện chuyên sâu về mật mã (`node-forge`, `crypto-js`), xử lý số lớn (`bignumber.js`), xử lý ảnh (`jimp`), và nhận diện văn bản (`tesseract.js`).

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của CyberChef được xây dựng dựa trên sự ẩn dụ về một **"Nhà bếp" (Kitchen)**, giúp tách biệt hoàn toàn giữa dữ liệu, logic xử lý và giao diện người dùng.

*   **Mô hình Dish - Recipe - Chef:**
    *   **Dish (Món ăn):** Đại diện cho dữ liệu. Nó không chỉ là chuỗi (string) mà có thể là `ArrayBuffer`, `JSON`, `File`, hoặc `Number`. Một điểm cực kỳ thông minh là `Dish.mjs` tự động xử lý việc chuyển đổi kiểu dữ liệu (Type Conversion) giữa các công đoạn.
    *   **Recipe (Công thức):** Là một danh sách các thao tác (Operations) mà người dùng đã chọn. Nó quản lý thứ tự thực thi và các tham số truyền vào.
    *   **Operation (Thao tác):** Mỗi công cụ (ví dụ: Base64, AES, XOR) là một lớp (class) độc lập kế thừa từ `Operation.mjs`. Điều này giúp việc thêm tính năng mới cực kỳ dễ dàng (chỉ cần tạo 1 file `.mjs` mới).
    *   **Chef (Đầu bếp):** Là bộ điều khiển trung tâm (`Chef.mjs`), chịu trách nhiệm kết hợp "Dữ liệu" và "Công thức" để tạo ra "Kết quả".
*   **Tính Mô-đun (Modularity):** Các Operation được chia thành các mô-đun (Modules). Hệ thống chỉ tải (Lazy load) các mô-đun cần thiết để giảm dung lượng tải trang ban đầu.
*   **Tách biệt UI/Logic:** Toàn bộ logic cốt lõi nằm trong `src/core/`, hoàn toàn không phụ thuộc vào giao diện DOM. Giao diện người dùng nằm trong `src/web/` chỉ đóng vai trò hiển thị và nhận tương tác.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Đa luồng với Web Workers:** Đây là kỹ thuật quan trọng nhất của CyberChef. Khi người dùng thực hiện một tác vụ nặng (Baking), nó được đẩy xuống `ChefWorker.js` chạy ở luồng nền. Điều này giúp giao diện không bị treo (freeze) ngay cả khi xử lý file dữ liệu hàng trăm MB.
*   **Cơ chế Highlight (Đánh dấu tương ứng):** CyberChef có khả năng ánh xạ vị trí (offset) của dữ liệu từ ô Input sang ô Output và ngược lại. Kỹ thuật này đòi hỏi mỗi Operation phải cài đặt phương thức `highlight()` để tính toán sự thay đổi vị trí dữ liệu.
*   **Tự động nhận diện (Magic Mode):** Sử dụng các đặc điểm của dữ liệu (Magic bytes, cấu trúc entropy) để đoán xem dữ liệu đang được mã hóa bằng phương pháp nào và gợi ý cho người dùng.
*   **Hệ thống Dish Type Conversion:** Cơ chế tự động chuyển đổi từ `ByteArray` sang `String` hoặc `ArrayBuffer` một cách linh hoạt nhờ vào lớp `Dish.mjs`. Điều này cho phép một Operation đầu ra là Byte nhưng Operation tiếp theo có thể nhận đầu vào là String mà không bị lỗi.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng đi của dữ liệu trong CyberChef diễn ra như sau:

1.  **Tiếp nhận (Input):** Người dùng nhập văn bản hoặc kéo thả file vào ô Input. Trình quản lý giao diện (`Manager.mjs`) ghi nhận sự thay đổi.
2.  **Lập kế hoạch (Recipe Building):** Người dùng kéo các Operation vào Recipe. Mỗi Operation đi kèm với các tham số (Ingredients - Nguyên liệu).
3.  **Thực thi (Baking):**
    *   `App.mjs` gửi một thông điệp tới `WorkerWaiter.mjs`.
    *   `WorkerWaiter` chuyển dữ liệu và cấu hình recipe sang `ChefWorker.js` (Web Worker).
    *   Tại đây, `Chef.mjs` sẽ khởi tạo một đối tượng `Dish` từ dữ liệu đầu vào.
    *   `Recipe.mjs` sẽ duyệt qua từng Operation trong danh sách.
    *   Với mỗi Operation, `Dish` sẽ tự kiểm tra xem kiểu dữ liệu hiện tại có phù hợp với yêu cầu của Operation đó không (ví dụ: AES cần `ArrayBuffer`, nhưng dữ liệu hiện tại đang là `String`). Nếu không, `Dish` sẽ tự chuyển đổi.
    *   Kết quả của Operation trước trở thành đầu vào của Operation sau.
4.  **Trình diễn (Presentation):** Sau khi hoàn thành các bước, kết quả cuối cùng được gửi ngược lại luồng chính (Main Thread).
5.  **Hiển thị (Output):** Dữ liệu được hiển thị lên ô Output. Nếu kết quả là định dạng đặc biệt (hình ảnh, HTML), CyberChef sẽ render tương ứng.

### Tổng kết
Dự án CyberChef là một ví dụ mẫu mực về **Kiến trúc hướng đối tượng (OOP)** và **Xử lý bất đồng bộ** trong JavaScript. Việc sử dụng Web Workers và mô hình "Nhà bếp" giúp nó trở thành một công cụ cực kỳ mạnh mẽ, mở rộng tốt và an toàn (vì dữ liệu không bao giờ rời khỏi máy khách).