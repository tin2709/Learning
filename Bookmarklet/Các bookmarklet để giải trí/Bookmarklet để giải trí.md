

## 1 Loop youtube ở 1 khoảng thời gian cho sẵn
### Bước 1: Copy đoạn mã chuyên dụng cho Bookmarklet
Tôi đã tinh chỉnh lại mã một chút để khi bạn bấm vào nút, nó sẽ hiện ra ô hỏi bạn muốn lặp từ giây bao nhiêu đến giây bao nhiêu.

**Hãy copy đoạn mã dưới đây:**

```javascript
javascript:(function(){var start=prompt("Nhập giây BẮT ĐẦU:","11");var end=prompt("Nhập giây KẾT THÚC:","254");if(start!==null && end!==null){function simpleLoop(s,e){var p=document.getElementById("movie_player");if(!p)return;var c=p.getCurrentTime();var t;if(c<s||c>=e){p.seekTo(s);t=e-s}else{t=e-c}console.log("Looping...");setTimeout(function(){simpleLoop(s,e)},t*1000)}simpleLoop(parseFloat(start),parseFloat(end))}})();
```

### Bước 2: Tạo Bookmark (Dấu trang)
Dựa trên cái ảnh bạn đang mở, hãy làm như sau:

1.  **Tên:** Đặt tên là `Lặp Video YT` (hoặc tên gì bạn thích).
2.  **URL (Thư mục/Địa chỉ):** Bạn xóa sạch cái link `https://www.youtube.com...` hiện tại đi.
3.  **Dán đoạn code** bạn vừa copy ở Bước 1 vào ô đó.
4.  Nhấn **Xong (Done)**.

### Bước 3: Cách sử dụng
Bất cứ khi nào bạn đang xem video trên YouTube:

1.  Nhìn lên thanh dấu trang (Bookmark bar) của trình duyệt.
2.  Bấm vào cái nút `Lặp Video YT` bạn vừa tạo.
3.  Một cái bảng hiện lên hỏi giây **Bắt đầu**, bạn nhập (ví dụ 11) -> OK.
4.  Một cái bảng hiện lên hỏi giây **Kết thúc**, bạn nhập (ví dụ 254) -> OK.
5.  Video sẽ tự động lặp trong khoảng đó cho đến khi bạn F5 lại trang.

### Mẹo nhỏ:
*   Nếu bạn không thấy thanh dấu trang, hãy nhấn `Ctrl + Shift + B` để nó hiện ra.
*   Mã này chỉ chạy trên trang web YouTube chính thức.
*   Để dừng lặp, bạn chỉ cần nhấn **F5** để tải lại trang.