

# 1 Tìm hiểu về API `moveBefore()` mới của DOM

API `moveBefore()` mới được Chrome công bố là một bổ sung tiềm năng cho cách chúng ta xử lý các phần tử trong DOM. Nó giúp các nhà phát triển dễ dàng định vị lại các phần tử trong khi vẫn giữ nguyên trạng thái của chúng, điều mà các phương pháp truyền thống gặp khó khăn.

## Vấn đề với cách di chuyển phần tử DOM truyền thống

Trong nhiều năm, khi cần di chuyển một phần tử DOM (ví dụ: từ container A sang container B), cách làm phổ biến là:

1.  Xóa phần tử đó khỏi vị trí hiện tại (ví dụ: dùng `removeChild`).
2.  Chèn phần tử đó vào vị trí mới (ví dụ: dùng `appendChild` hoặc `insertBefore`).

Cách tiếp cận "xóa và chèn" này đơn giản nhưng gây ra ba vấn đề chính:

1.  **Mất trạng thái (State Loss):** Trạng thái nội bộ của phần tử thường bị reset. Ví dụ: một animation CSS đang chạy sẽ bị khởi động lại, nội dung đang phát của một `<iframe>` (như video YouTube) sẽ bị dừng hoặc reset, hoặc vị trí cuộn của một div sẽ trở về đầu.
2.  **Hiệu suất (Performance Issues):** Việc xóa và chèn có thể kích hoạt các quá trình tính toán lại bố cục (reflow) và vẽ lại (repaint) của trình duyệt. Với các ứng dụng lớn hoặc các thao tác di chuyển thường xuyên, điều này có thể gây ra hiện tượng "giật" (jank) hoặc chậm trễ giao diện.
3.  **Code phức tạp (Verbose Code):** Để tránh mất trạng thái hoặc giảm thiểu vấn đề hiệu suất, nhà phát triển thường phải viết thêm code phức tạp (workaround) để lưu trữ và khôi phục trạng thái hoặc tối ưu hóa các thao tác DOM.

## Giải pháp: API `moveBefore()`

API `moveBefore()` được giới thiệu để khắc phục những hạn chế này. Thay vì xóa và chèn lại, `moveBefore()` thực hiện một thao tác **"di chuyển nguyên tử" (atomic move)**. Điều này có nghĩa là phần tử được di chuyển trực tiếp từ vị trí cũ sang vị trí mới *mà không bị loại bỏ hoàn toàn khỏi cây DOM trong quá trình*, nhờ đó **giữ nguyên được trạng thái của nó**.

## Cách hoạt động

Cú pháp của `moveBefore()` rất giống với `insertBefore()`:

```javascript
parentNode.moveBefore(nodeToMove, referenceNode);
```

Giải thích các tham số:

*   `parentNode`: Nút cha đích (destination parent node). Đây là container mà bạn muốn `nodeToMove` được di chuyển vào bên trong. Nó phải là một nút có khả năng chứa các nút con (như `<div>`, `<body>`, v.v.).
*   `nodeToMove`: Phần tử (Node) mà bạn muốn di chuyển. Phần tử này có thể đang nằm trong DOM (gắn với một nút cha khác) hoặc đang ở trạng thái độc lập (detached). **Quan trọng: Trạng thái của nút này được bảo toàn khi sử dụng `moveBefore()`.**
*   `referenceNode`: Nút tham chiếu. Tham số này chỉ định vị trí cụ thể mà `nodeToMove` sẽ được chèn vào trong danh sách con của `parentNode`.
    *   Nếu `referenceNode` là một nút con **trực tiếp** của `parentNode`, `nodeToMove` sẽ được chèn **ngay trước** nút `referenceNode` đó.
    *   Nếu `referenceNode` là `null`, `nodeToMove` sẽ được thêm vào **cuối cùng** danh sách các nút con của `parentNode` (tương tự như `appendChild`).

## Đặc điểm của `moveBefore()`

*   **Di chuyển nguyên tử:** Không phải là xóa rồi chèn, giúp bảo toàn trạng thái.
*   **Cú pháp tương tự `insertBefore()`:** Dễ học và dễ nhớ đối với các nhà phát triển quen thuộc với các thao tác DOM truyền thống.
*   **Xử lý lỗi:** Ném ra `DOMException` nếu `referenceNode` không hợp lệ (không phải con trực tiếp của `parentNode` và không phải `null`) hoặc nếu `nodeToMove` không thể di chuyển đến `parentNode` (ví dụ: `nodeToMove` là tổ tiên của `parentNode`).

## Ví dụ thực tế: Chuyển đổi layout video

Hãy xem xét kịch bản một video player (được nhúng bằng `<iframe>`) có thể chuyển đổi giữa chế độ toàn màn hình và chế độ chia màn hình (cùng với khu vực ghi chú).

Với phương pháp truyền thống (`appendChild` hoặc `insertBefore`), khi di chuyển `<iframe>` giữa hai container khác nhau, video đang phát sẽ bị reset. Người dùng sẽ mất vị trí đang xem.

Với `moveBefore()`, bạn có thể di chuyển `<iframe>` giữa các container khác nhau mà video vẫn **tiếp tục phát** mà không bị gián đoạn, mang lại trải nghiệm người dùng liền mạch.

*(Xem code ví dụ đầy đủ trong bài viết gốc để thấy sự khác biệt giữa hai phương pháp)*

Đoạn code minh họa cách sử dụng `moveBefore()` (so với `appendChild` làm fallback):

```javascript
const videoIframe = document.getElementById('video');
const fullScreenContainer = document.getElementById('full-screen-container');
const splitScreenContainer = document.getElementById('split-screen-container');
const splitVideoWrapper = splitScreenContainer.querySelector('.video-wrapper'); // Lấy wrapper bên trong split container

let isFullScreen = true;

function toggleLayout() {
    if (isFullScreen) {
        // Chuyển sang chế độ chia màn hình
        fullScreenContainer.style.display = 'none';
        splitScreenContainer.style.display = 'block';
        // ... (thay đổi kích thước iframe, hiển thị notesContainer)

        // Di chuyển iframe
        if ('moveBefore' in Element.prototype) {
            // Sử dụng moveBefore() nếu được hỗ trợ
            splitVideoWrapper.moveBefore(videoIframe, null); // Di chuyển vào splitVideoWrapper, ở cuối
        } else {
            // Fallback: Sử dụng appendChild()
            splitVideoWrapper.appendChild(videoIframe);
        }

    } else {
        // Chuyển sang chế độ toàn màn hình
        fullScreenContainer.style.display = 'block';
        splitScreenContainer.style.display = 'none';
        // ... (thay đổi kích thước iframe, ẩn notesContainer)

        // Di chuyển iframe
        const fullVideoWrapper = fullScreenContainer.querySelector('.video-wrapper'); // Lấy wrapper bên trong full screen container
         if ('moveBefore' in Element.prototype) {
            // Sử dụng moveBefore()
            fullVideoWrapper.moveBefore(videoIframe, null); // Di chuyển vào fullVideoWrapper, ở cuối
        } else {
            // Fallback: Sử dụng appendChild()
            fullVideoWrapper.appendChild(videoIframe);
        }
    }
    isFullScreen = !isFullScreen;
}
```

## Hỗ trợ trình duyệt

Tính đến tháng 4 năm 2025, `moveBefore()` được hỗ trợ trên **Chrome 133+**. Safari và Firefox đã bày tỏ sự quan tâm nhưng chưa có bản hỗ trợ chính thức.

Do đó, khi sử dụng API này trong môi trường production, việc kiểm tra hỗ trợ và cung cấp phương án dự phòng (fallback) bằng `appendChild` hoặc `insertBefore` là **rất cần thiết**:

```javascript
if ('moveBefore' in Element.prototype) {
    // Sử dụng moveBefore() API
    parentNode.moveBefore(nodeToMove, referenceNode);
} else {
    // Trình duyệt không hỗ trợ, sử dụng phương pháp truyền thống làm fallback
    parentNode.insertBefore(nodeToMove, referenceNode); // Hoặc appendChild nếu referenceNode là null
}
```

## Kết luận

API `moveBefore()` là một bước tiến quan trọng trong việc đơn giản hóa và cải thiện hiệu suất, trải nghiệm khi di chuyển các phần tử trong DOM, đặc biệt là các phần tử có trạng thái phức tạp

---