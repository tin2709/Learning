Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **FleshNote IDE**, dưới đây là bản phân tích chi tiết về hệ thống:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

FleshNote được thiết kế theo mô hình **Hybrid Desktop Application**, kết hợp hiệu suất tính toán ngôn ngữ của Python và sự linh hoạt của giao diện Web:

*   **Frontend Framework:** **React** phối hợp với **Vite** để đảm bảo tốc độ phản hồi UI cực nhanh.
*   **Editor Engine:** **TipTap (dựa trên ProseMirror)**. Đây là thành phần quan trọng nhất, cho phép can thiệp sâu vào cấu trúc văn bản (Node/Mark) để chèn các siêu dữ liệu (metadata) như liên kết thực thể hoặc mã thời gian.
*   **Backend API:** **FastAPI (Python 3.13)**. Đảm nhận việc xử lý logic nghiệp vụ phức tạp, quản lý tệp tin và các tác vụ nặng về CPU.
*   **Database:** **SQLite (WAL Mode)**. Mỗi dự án là một file cơ sở dữ liệu riêng biệt, cho phép truy vấn quan hệ phức tạp giữa nhân vật, sự kiện và kiến thức mà các file text thuần túy không làm được.
*   **NLP & Linguistic Analysis:** **spaCy** (để trích xuất thực thể NER, phân tích cú pháp), **NLTK** (từ điển WordNet cho từ đồng nghĩa) và **Hunspell** (kiểm tra lỗi chính tả).
*   **Desktop Shell:** **Electron**. Cung cấp khả năng truy cập hệ thống tệp tin cục bộ, quản lý cửa sổ và tích hợp đa nền tảng (Windows, macOS, Linux).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của FleshNote tập trung vào việc giải quyết bài toán: **Làm sao để quản lý hàng triệu từ và hàng nghìn kết nối lore mà vẫn giữ được sự mượt mà?**

*   **Kiến trúc IPC 3 lớp (3-Layer IPC):**
    *   **Layer 1:** React UI gọi các hàm API qua `contextBridge`.
    *   **Layer 2:** Electron Main Process nhận yêu cầu và chuyển tiếp qua HTTP tới Backend Python.
    *   **Layer 3:** Python FastAPI thực thi logic và truy vấn SQLite.
    *   *Lợi ích:* Cô lập hoàn toàn logic xử lý văn bản nặng khỏi luồng xử lý giao diện, giúp UI không bao giờ bị "treo".
*   **Lưu trữ Hybrid (Markdown-Entity Hybrid):** Prose (văn xuôi) được lưu thành các file `.md` riêng lẻ trên đĩa để người dùng có thể mở bằng bất kỳ trình đọc nào, nhưng các quan hệ logic được lưu trong SQLite.
*   **Hệ thống Thời gian kép (Dual-Timeline):** Tách biệt "Narrative Time" (thứ tự xuất hiện trong truyện) và "World Time" (thời gian thực tế trong lịch sử thế giới). Đây là tư duy cốt lõi dành cho các nhà văn viết truyện không tuyến tính (flashback, xuyên không).
*   **Thiết kế Fail-Safe NLP:** Các model ngôn ngữ nặng (hàng trăm MB) không được đóng gói sẵn vào bộ cài. Hệ thống sẽ tự động tải (dynamic download) về thư mục `AppData` của người dùng khi cần, giúp bộ cài cực kỳ tinh gọn.

---

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **TipTap Custom Marks:** Sử dụng các "Mark" tùy chỉnh để đánh dấu văn bản mà không làm hỏng dữ liệu gốc. Ví dụ: `{{char:5|Sophia}}` trên đĩa sẽ được TipTap render thành một `<span>` có màu sắc và thuộc tính tương tác trong trình soạn thảo.
*   **Debounced Autonomous Save:** Hệ thống tự động lưu sau 500ms người dùng ngừng gõ. Trong quá trình lưu, backend sẽ thực hiện "Refactoring" lại văn bản, quét các thẻ thực thể và cập nhật bảng `entity_appearances` (ma trận xuất hiện) theo thời gian thực.
*   **Epistemic Filtering (Lọc tri thức):** Sử dụng các câu lệnh SQL phức tạp để lọc thông tin nhân vật dựa trên POV (Point of View). Hệ thống sẽ ẩn/hiện các thông tin "Sự thật" hoặc "Bí mật" tùy thuộc vào việc nhân vật đó đã biết thông tin đó ở chương nào chưa.
*   **Hierarchical Weather Inheritance:** Kỹ thuật kế thừa thuộc tính trong môi trường. Nếu một địa điểm con (ví dụ: "Phòng ngủ") không có dữ liệu thời tiết, hệ thống sẽ đệ quy truy vấn lên địa điểm cha ("Lâu đài") để lấy thông tin.
*   **Fuzzy Radius Search:** Giải quyết vấn đề lệch offset giữa TipTap HTML và Plain Text. Khi Janitor gợi ý sửa một từ, frontend sẽ tìm kiếm trong một bán kính nhất định quanh vị trí được chỉ định để re-anchor (neo lại) chính xác vị trí cần highlight.

---

### 4. Luồng hoạt động hệ thống (Operational Workflow)

1.  **Khởi động (Bootstrapping):** Electron khởi chạy -> Kích hoạt tiến trình Python ngầm -> Chờ API sẵn sàng -> Tải cấu hình dự án từ SQLite.
2.  **Soạn thảo (Editing Loop):**
    *   Người dùng nhập liệu.
    *   TipTap xử lý định dạng rich-text cục bộ.
    *   Hệ thống gán nhãn thực thể (@mention) tức thì qua menu gợi ý.
3.  **Phân tích ngầm (Janitor Analysis):** Khi người dùng ngừng gõ 10 giây -> Toàn bộ văn bản chương được gửi tới Python -> spaCy phân tích câu, phát hiện giọng thụ động, trạng từ yếu, lặp từ hoặc thiếu miêu tả cảm giác (Sensory analysis) -> Trả về danh sách card gợi ý.
4.  **Chế độ Sprint (Sprint Execution):** Người dùng bật "Kamikaze Mode" -> Frontend kích hoạt Timer -> Nếu tốc độ gõ giảm xuống dưới ngưỡng, UI sẽ chuyển sang đỏ và bắt đầu thực hiện lệnh xóa văn bản qua API.
5.  **Xuất bản (Exporting):** Người dùng chọn định dạng -> Stage 1: Python quét văn bản, chuyển đổi các thẻ thực thể thành chú thích (footnotes) hoặc text thuần -> Stage 2: Sử dụng `python-docx` hoặc `weasyprint` để render file cuối cùng với lề và gutter chuẩn in ấn.

**Tổng kết:** FleshNote IDE là một sự kết hợp mạnh mẽ giữa các công cụ xử lý ngôn ngữ tự nhiên hiện đại và tư duy quản lý dữ liệu chặt chẽ, được tối ưu hóa riêng cho việc sáng tác văn học quy mô lớn.