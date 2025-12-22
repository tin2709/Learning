
## Happy-dom

### 1. Happy-dom là gì?
**Happy-dom** là một thư viện JavaScript dùng để mô phỏng môi trường trình duyệt (Web Browser) chạy hoàn toàn bên trong Node.js. Nó tạo ra các đối tượng giả lập như `window`, `document`, `HTMLElement`... giúp code Frontend có thể hoạt động mà không cần mở trình duyệt thật.

### 2. Nó giải quyết vấn đề gì?
Thông thường, Node.js không hiểu được các thẻ HTML hay các sự kiện như "click". Để kiểm thử (test) giao diện, bạn thường phải dùng trình duyệt thật (rất chậm) hoặc các thư viện giả lập cũ như JSDOM (nặng nề). Happy-dom giúp chạy các bản test UI với tốc độ cực nhanh và tốn ít tài nguyên hơn.

### 3. Các tính năng chính:
*   **Siêu tốc độ:** Được tối ưu hóa để khởi tạo và thực thi các cấu trúc DOM nhanh hơn nhiều so với các đối thủ cạnh tranh.
*   **Nhẹ nhàng:** Tiêu tốn cực ít RAM và CPU, cho phép chạy hàng ngàn bản test cùng lúc mà không gây treo máy.
*   **Hỗ trợ đầy đủ Web API:** Mô phỏng tốt các tính năng hiện đại như Fetch API, Custom Elements, và các sự kiện trình duyệt phức tạp.
*   **Tương thích tuyệt vời:** Hoạt động hoàn hảo với các khung kiểm thử hiện đại như **Vitest** hoặc **Jest**.

### 4. Cách sử dụng phổ biến
Thường xuất hiện trong cấu hình của các công cụ test (như Vitest) để chỉ định môi trường chạy:
*   `test: { environment: 'happy-dom' }` (Trong file vite.config.ts).

### Lưu ý phân biệt:
Khác với **Selenium** hay **Playwright** (chạy trên trình duyệt thật để test E2E), Happy-dom chỉ là một **môi trường giả lập** dùng để Unit Test logic của các component giao diện.

**Tóm lại:** Đây là một "trình duyệt ảo" siêu nhẹ, giúp lập trình viên kiểm tra code giao diện ngay trong terminal với tốc độ cao nhất.

