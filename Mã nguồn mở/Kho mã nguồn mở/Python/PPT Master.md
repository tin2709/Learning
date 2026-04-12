Dưới đây là bản phân tích chuyên sâu về hệ thống **PPT Master** dựa trên mã nguồn và tài liệu bạn cung cấp. Đây là một dự án kết hợp tinh tế giữa kỹ thuật Prompt Engineering (AI) và kỹ thuật chuyển đổi đồ họa (Graphics Engineering).

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ tập trung vào xử lý văn bản và chuyển đổi vector:

*   **Ngôn ngữ lập trình:** **Python (85.8%)** là ngôn ngữ chủ đạo, xử lý toàn bộ logic chuyển đổi và hậu xử lý. **JavaScript/CJS** được dùng cho các tác vụ đặc thù như cào dữ liệu từ WeChat.
*   **Chuyển đổi tài liệu:** Sử dụng **Pandoc** (cho DOCX/EPUB), **PyMuPDF** (cho PDF) để "Markdown hóa" mọi nguồn đầu vào, tạo ra một cấu trúc dữ liệu thuần văn bản mà AI dễ dàng phân tích nhất.
*   **Xử lý Đồ họa Vector:** 
    *   **SVG (Scalable Vector Graphics):** Đóng vai trò là định dạng trung gian (Pivot format). AI sẽ tạo ra SVG vì đây là ngôn ngữ thiết kế vector phổ biến, dễ học và có hệ tọa độ tuyệt đối.
    *   **DrawingML:** Ngôn ngữ XML nội bộ của Microsoft Office. Dự án không yêu cầu AI viết DrawingML trực tiếp (vì quá phức tạp) mà dùng script để chuyển từ SVG sang.
*   **Thư viện PPTX:** `python-pptx` làm hạt nhân để đóng gói file, nhưng được mở rộng bằng logic tùy chỉnh để ghi đè XML (DrawingML) nhằm tạo ra các hình khối "natively editable" (có thể chỉnh sửa gốc).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của PPT Master không phải là một ứng dụng "One-click" thông thường mà là một **"Collaborative AI Pipeline"** (Đường ống cộng tác AI):

*   **Pipeline phân vai (Role-based Pipeline):** Hệ thống chia quy trình thành 3 thực thể logic:
    1.  **Strategist (Chiến lược gia):** Chịu trách nhiệm phân tích nội dung, xác định cấu trúc (SCQA - Situation, Complication, Question, Answer) và lập spec thiết kế.
    2.  **Image Generator:** Phụ trách thẩm mỹ thị giác thông qua việc tối ưu Prompt cho các nền tảng như Gemini, Midjourney.
    3.  **Executor (Người thực thi):** Chuyển spec thành mã SVG và soạn thảo Speaker Notes.
*   **Pivot Format Architecture:** Kiến trúc lấy SVG làm trung tâm. Mọi yếu tố từ biểu đồ, icon đến layout đều được quy về tọa độ SVG. Điều này cho phép hệ thống tách rời việc "Thiết kế visual" (do AI làm) và "Đóng gói kỹ thuật" (do script làm).
*   **Strict Serial Discipline:** Tư duy kiến trúc yêu cầu thực thi tuần tự nghiêm ngặt (Serial Execution). Đầu ra của bước này là đầu vào không thể thiếu của bước sau, giúp kiểm soát lỗi và duy trì tính nhất quán về phong cách (Visual Cohesion).

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **SVG-to-DrawingML Mapping:** Đây là kỹ thuật khó nhất. Script xử lý việc ánh xạ các thẻ SVG (`<path>`, `<rect>`, `<circle>`) sang các thẻ DrawingML tương ứng (`<a:custGeom>`, `<a:prstGeom>`). Nó xử lý cả các thuộc tính phức tạp như `linearGradient`, `radialGradient` và `opacity`.
*   **Marker & ClipPath Translation:** 
    *   Chuyển đổi `marker-end` (mũi tên trong SVG) thành thuộc tính đầu mũi tên bản xứ của PowerPoint (`<a:tailEnd>`).
    *   Chuyển đổi `clipPath` trên thẻ `<image>` thành `Custom Geometry` của Office để tạo ra các avatar tròn hoặc khung ảnh bo góc mà không cần cắt ảnh thủ công.
*   **Icon Placeholder Mechanism:** Sử dụng cơ chế thẻ `<use data-icon="...">`. Kỹ thuật này giúp AI chỉ cần gọi tên icon, còn script hậu xử lý sẽ tự động tra cứu trong thư viện 6700+ vector icon và nhúng mã vào file cuối cùng.
*   **Dynamic Layout Calculation:** Script `analyze_images.py` tự động tính toán tỷ lệ ảnh gốc để gợi ý layout (Chia 50/50, chia trên-dưới) nhằm tránh hiện tượng ảnh bị kéo dãn khi đưa vào slide.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình đi từ "Dữ liệu thô" đến "Sản phẩm có thể chỉnh sửa":

1.  **Conversion:** Tài liệu nguồn (PDF/URL...) → Markdown (Văn bản có cấu trúc).
2.  **Initialization:** Tạo project, chọn Template hoặc "Free Design".
3.  **Eight Confirmations:** AI Strategist trình bày 8 điểm xác nhận (Khổ giấy, đối tượng, màu sắc...) và **DỪNG LẠI (Blocking)** để chờ người dùng xác nhận.
4.  **Prompt Engineering:** Image Generator tạo file `image_prompts.md` và sinh ảnh AI (nếu cần).
5.  **SVG Generation:** Executor vẽ từng slide thành SVG. Quan trọng: AI vẽ slide theo kiểu "tọa độ tuyệt đối" (Absolute Positioning).
6.  **Logic Generation:** AI viết lời dẫn cho diễn giả đồng bộ với nội dung slide.
7.  **Post-processing Pipeline (3 bước rời):**
    *   *Step 7.1:* Chia nhỏ lời dẫn.
    *   *Step 7.2:* **Finalize SVG** (Bước quan trọng nhất: Nhúng icon, cắt ảnh, làm phẳng text).
    *   *Step 7.3:* **Export** (Chuyển SVG thành DrawingML và đóng gói vào file `.pptx`).

### Tổng kết
**PPT Master** khác biệt hoàn toàn với các công cụ tạo PPT bằng AI khác (thường chỉ sinh ra ảnh phẳng chèn vào slide) nhờ vào việc **biên dịch (compile)** các bản vẽ thiết kế của AI thành các đối tượng đồ họa gốc của Microsoft Office. Điều này biến AI từ một "người dán ảnh" thành một "nhà thiết kế đồ họa" thực thụ có khả năng giao tiếp với các phần mềm văn phòng thông qua mã nguồn.