
Dưới đây là cách nó hoạt động trên các trang web khác và những gì bạn có thể làm:

### 1. Nó hoạt động như thế nào trên các trang web khác?
Mỗi userscript đều có một phần khai báo ở đầu (gọi là Metadata). Bạn hãy để ý dòng `@match`. 
* Nếu script có dòng `// @match https://www.facebook.com/*`, nó sẽ chỉ chạy trên Facebook.
* Nếu script có dòng `// @match *://*/*`, nó sẽ chạy trên **tất cả mọi trang web** bạn truy cập.

### 2. Các ví dụ phổ biến bạn có thể tìm thấy:
Bạn có thể tìm kiếm userscript cho hầu hết các trang web lớn hiện nay:

*   **YouTube:** Tự động bỏ qua quảng cáo, thêm nút tải video, thay đổi giao diện rạp phim, hoặc hiển thị lại số lượt Dislike (đã bị ẩn).
*   **Facebook:** Ẩn các bài viết quảng cáo (Sponsored), dọn dẹp bảng tin (Newsfeed) cho gọn hơn.
*   **Google:** Thay đổi giao diện tìm kiếm, tự động chuyển sang trang tiếp theo khi cuộn chuột xuống dưới cùng.
*   **Trang truyện/phim:** Tự động tải tất cả các chương truyện vào một trang để đọc liền mạch, hoặc ẩn các popup quảng cáo khó chịu trên trang phim lậu.
*   **Tất cả các web:** Ép mọi trang web phải hiển thị **Chế độ tối (Dark Mode)** dù trang đó không hỗ trợ.

### 3. Tìm các script này ở đâu?
Bạn không cần phải tự viết code. Có những "kho" script khổng lồ do cộng đồng đóng góp. Bạn chỉ cần vào các trang này, tìm tên trang web bạn muốn thay đổi và nhấn **Install**:

1.  **Greasy Fork (Phổ biến nhất):** [greasyfork.org](https://greasyfork.org/vi) (Có hỗ trợ tiếng Việt).
2.  **OpenUserJS:** [openuserjs.org](https://openuserjs.org/).

### 4. Bạn có thể tự viết script không?
Hoàn toàn có thể. Nếu bạn biết một chút về **JavaScript** và **CSS**, bạn có thể tự tạo script để:
*   Tự động điền form trên một web nội bộ của công ty.
*   Ẩn đi một cái nút hoặc một đoạn văn bản vô dụng trên một trang web bạn hay vào.
*   Thay đổi màu sắc trang web cho hợp mắt hơn.

**Tóm lại:** Tampermonkey là một "cây đũa thần" cho trình duyệt. Chỉ cần tìm được đúng tập lệnh (userscript) phù hợp, bạn có thể biến đổi bất kỳ trang web nào theo ý thích cá nhân.