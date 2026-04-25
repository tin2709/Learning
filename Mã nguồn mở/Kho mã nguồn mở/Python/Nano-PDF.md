Dựa trên mã nguồn và tài liệu của kho lưu trữ **Nano-PDF Editor**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình hoạt động của công cụ này:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Công cụ này là một sự kết hợp giữa các thư viện xử lý tài liệu truyền thống và mô hình AI đa phương thức tiên tiến nhất:

*   **Google Gemini 3 Pro Image ("Nano Banana"):** Đây là trung tâm của hệ thống. Khác với các mô hình chỉ tạo văn bản, mô hình này cho phép nhập cả hình ảnh + văn bản và xuất ra hình ảnh mới. Nó xử lý việc hiểu bố cục slide, thay đổi nội dung đồ họa và duy trì tính thẩm mỹ.
*   **Poppler (pdf2image & pdftotext):** Được sử dụng để "chụp ảnh" các trang PDF thành định dạng PIL Image để AI có thể "nhìn" thấy, đồng thời trích xuất văn bản thô để làm ngữ cảnh (context).
*   **Tesseract OCR:** Đóng vai trò quan trọng trong việc "Re-hydration" (tái cấp ẩm). Sau khi AI tạo ra một bức ảnh slide mới, Tesseract sẽ quét lại bức ảnh đó để tạo ra một lớp văn bản ẩn (searchable text layer).
*   **PyPDF:** Thư viện xử lý cấu trúc file PDF, dùng để cắt ghép, chèn hoặc thay thế các trang đã được AI chỉnh sửa vào file gốc mà không làm hỏng cấu trúc các trang còn lại.
*   **Typer:** Framework giúp xây dựng giao diện dòng lệnh (CLI) chuyên nghiệp và dễ sử dụng.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Nano-PDF đi theo hướng **"Hybrid Pipeline"** (Đường ống hỗn hợp):

*   **Tính mô-đun (Modularity):** Tách biệt rõ rệt giữa `ai_utils.py` (giao tiếp với LLM) và `pdf_utils.py` (thao tác file vật lý). Điều này cho phép dễ dàng thay đổi mô hình AI trong tương lai mà không cần viết lại logic xử lý PDF.
*   **Xử lý song song (Concurrency):** Vì việc gọi API AI tạo hình ảnh thường mất nhiều thời gian, tác giả sử dụng `concurrent.futures.ThreadPoolExecutor` để xử lý tối đa 10 trang cùng lúc. Đây là tư duy tối ưu hiệu suất quan trọng cho một công cụ CLI.
*   **Triết lý "Không phá hủy" (Non-destructive):** Thay vì biến PDF thành một tệp ảnh phẳng (flattened), kiến trúc cố gắng bảo toàn lớp văn bản có thể chọn được (selectable text) thông qua OCR, giúp tài liệu sau khi sửa vẫn giữ được công năng chuyên nghiệp.

### 3. Các kỹ thuật chính (Key Techniques)
Có 3 kỹ thuật "bí mật" giúp công cụ này vượt trội hơn các trình chỉnh sửa ảnh thông thường:

*   **OCR Re-hydration (Tái cấp ẩm văn bản):** Đây là kỹ thuật biến ảnh đầu ra của AI trở lại thành PDF có lớp văn bản. Bằng cách chạy `pytesseract.image_to_pdf_or_hocr`, công cụ "nhúng" các tọa độ chữ vào slide, cho phép người dùng bôi đen và copy văn bản trên slide do AI tạo ra.
*   **Visual Style In-Context Learning:** Khi người dùng cung cấp `--style-refs`, công cụ sẽ gửi các ảnh slide mẫu vào prompt của Gemini. Điều này giúp AI học được font chữ, bảng màu và phong cách thiết kế của bộ slide hiện tại để tạo ra slide mới trông đồng bộ.
*   **Spatial Context via Layout Text:** Sử dụng lệnh `pdftotext -layout` để trích xuất văn bản nhưng giữ nguyên vị trí không gian tương đối. Văn bản này được đưa vào thẻ `<page-n>` giúp LLM hiểu được cấu trúc nội dung hiện có trên trang trước khi thực hiện chỉnh sửa.

### 4. Luồng hoạt động (Operational Workflow)
Quy trình thực thi một lệnh `edit` hoặc `add` diễn ra như sau:

1.  **Phân tích & Kiểm tra:** CLI phân tích các cặp `Trang - Yêu cầu`. Kiểm tra xem hệ thống đã cài Poppler và Tesseract chưa.
2.  **Chuẩn bị ngữ cảnh:**
    *   Trích xuất văn bản toàn bộ tài liệu (nếu bật `--use-context`).
    *   Render các trang cần sửa thành ảnh 4K/2K.
    *   Render các trang tham chiếu (style refs) thành ảnh.
3.  **Xử lý AI (Song song):**
    *   Gửi tổ hợp: `Ảnh gốc + Ảnh tham chiếu + Văn bản ngữ cảnh + Yêu cầu người dùng` tới Gemini API.
    *   Nhận về ảnh đã chỉnh sửa và phản hồi văn bản từ AI.
4.  **Hậu xử lý (Post-processing):**
    *   Dùng Tesseract chuyển ảnh AI thành một file PDF tạm thời có chứa lớp OCR.
    *   Dùng PyPDF để co dãn (scale) trang mới này cho khớp chính xác với kích thước trang gốc.
5.  **Hợp nhất (Stitching):** Ghi đè các trang mới vào vị trí cũ trong file PDF gốc và xuất ra file kết quả cuối cùng.

**Tóm lại:** Nano-PDF không chỉ là một trình wrapper API, mà là một giải pháp hoàn chỉnh giải quyết bài toán khó nhất của PDF: **Chỉnh sửa nội dung đồ họa mà vẫn giữ được tính chất kỹ thuật của tài liệu.**