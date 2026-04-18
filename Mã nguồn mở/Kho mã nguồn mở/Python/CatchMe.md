Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chuyên sâu về **CatchMe** - một hệ thống ghi chép và truy xuất dấu chân kỹ thuật số (digital footprint) thế hệ mới.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technologies)

CatchMe không đi theo lối mòn của các hệ thống RAG (Retrieval-Augmented Generation) thông thường. Các công nghệ chủ chốt bao gồm:

*   **Vectorless Memory (Bộ nhớ không vector):** Đây là điểm khác biệt lớn nhất. Thay vì sử dụng Vector Database và Embedding (vốn tốn tài nguyên và mất ngữ cảnh thời gian), CatchMe sử dụng **Hierarchical Activity Tree** (Cây hoạt động phân cấp).
*   **Multimodal LLM Integration:** Sử dụng LLM đa phương thức (Vision + Text) để hiểu nội dung màn hình. L0 (Action level) sử dụng Vision để tóm tắt các cụm thao tác chuột, trong khi các level cao hơn sử dụng Text LLM.
*   **SQLite + FTS5:** Sử dụng SQLite làm backend lưu trữ cục bộ, kết hợp với module FTS5 (Full-Text Search) để tìm kiếm từ khóa cực nhanh trên dữ liệu thô (keystrokes, window titles).
*   **Platform-native APIs:**
    *   **macOS:** Sử dụng `Quartz`, `ScreenCaptureKit`, và `Accessibility API (AX)` để bắt sự kiện ở mức hệ thống.
    *   **Windows:** Sử dụng `pywin32`, `comtypes` và `UI Automation` để theo dõi cửa sổ và văn bản.
*   **Trafilatura & PyMuPDF:** Các thư viện chuyên dụng để trích xuất nội dung từ URL và file PDF mà người dùng đang xem để làm giàu ngữ cảnh (context enrichment).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của CatchMe được thiết kế theo nguyên lý **"Bottom-up Abstraction, Top-down Retrieval"** (Trừu tượng hóa từ dưới lên, Truy xuất từ trên xuống):

#### A. Cấu trúc cây 5 tầng (Hierarchical Activity Tree):
1.  **Day (Ngày):** Nút gốc.
2.  **Session (Phiên):** Phân tách dựa trên thời gian nhàn rỗi (Idle timeout - mặc định 5 phút).
3.  **App (Ứng dụng):** Nhóm theo tiến trình (Chrome, VS Code, Slack...).
4.  **Location (Vị trí):** Nhóm theo URL cụ thể hoặc đường dẫn file.
5.  **Action (Hành động):** Các cụm (cluster) sự kiện bàn phím/chuột trong một khoảng thời gian ngắn (mặc định gap 3s).

#### B. Cơ chế "Event-Driven" thay vì "Polling":
Thay vì chụp ảnh màn hình định kỳ (gây tốn đĩa và CPU), CatchMe chỉ chụp ảnh khi có sự kiện chuột hoặc thay đổi cửa sổ đáng chú ý. Điều này giúp hệ thống cực kỳ nhẹ (~0.2GB RAM).

#### C. Tách biệt Backend (Awake) và Frontend (Web):
Hệ thống chạy một daemon ghi dữ liệu liên tục (`catchme awake`) và một web server riêng biệt để truy vấn dữ liệu. Giao tiếp giữa hai bên thông qua file JSONL và SQLite trong thư mục `data/`.

---

### 3. Kỹ thuật Lập trình Chính (Key Programming Techniques)

*   **Async Summary Queue (Hàng đợi tóm tắt bất đồng bộ):**
    *   Trong `summary_queue.py`, tác giả sử dụng `PriorityQueue` và `ThreadPoolExecutor`.
    *   **Chiến lược tầng bậc:** Chỉ khi các nút con (L0) được tóm tắt xong, nút cha (L1) mới được đưa vào hàng đợi. Điều này đảm bảo tóm tắt cấp cao luôn có đủ thông tin từ cấp thấp.
*   **IME-Aware Keyboard Logging:**
    *   Kỹ thuật xử lý tiếng Việt/Trung/Nhật: Thay vì ghi từng phím, hệ thống sử dụng `AXMarkedTextRange` (Mac) hoặc `UIA Value Pattern` (Win) để lấy đoạn văn bản đã "commit" cuối cùng, tránh rác dữ liệu từ quá trình gõ dấu.
*   **Temporal Clustering (Gom cụm thời gian):**
    *   Sử dụng thuật toán `cluster_events` trong `filter.py` để nhóm các sự kiện thô thành "Hành động". Nó xử lý thông minh các trạng thái như "đang cuộn chuột" (scroll session) để không cắt ngang hành động của người dùng.
*   **Tree-based Reasoning (Suy luận dựa trên cây):**
    *   Trong `retrieve.py`, thay vì tìm kiếm tương đồng (similarity search), CatchMe yêu cầu LLM thực hiện một vòng lặp: **Select (Chọn nhánh) -> Evaluate (Đánh giá nội dung) -> Deeper (Đi sâu xuống) hoặc Answer (Trả lời)**.

---

### 4. Luồng hoạt động của Hệ thống (System Workflow)

1.  **Giai đoạn Ghi (Record):**
    *   `Engine` khởi động các `Recorders`.
    *   Dữ liệu thô (Raw Events) được đẩy vào `Queue` và ghi batch vào SQLite để tối ưu I/O.
    *   Đồng thời, `Organizer` theo dõi các "biên giới" (boundary) như chuyển cửa sổ hoặc máy treo.
2.  **Giai đoạn Tổ chức (Organize):**
    *   `Organizer` định kỳ xây dựng lại cây hoạt động từ dữ liệu SQLite thô.
    *   Các nút đã "đóng" (người dùng đã chuyển sang hành động khác) được đẩy vào `SummaryQueue`.
    *   LLM chạy ngầm để tóm tắt từng nút từ dưới lên trên.
3.  **Giai đoạn Truy xuất (Retrieve):**
    *   Người dùng hỏi: "Sáng nay tôi đã nghiên cứu gì về AI?"
    *   LLM bước 1: Đọc tóm tắt cấp **Day/Session** để xác định khoảng thời gian.
    *   LLM bước 2: Đọc tóm tắt cấp **App/Location** để tìm các tài liệu liên quan.
    *   LLM bước 3: Nếu cần chi tiết, nó sẽ trích xuất nội dung thô (keystrokes) hoặc ảnh chụp màn hình tại nút **Action** tương ứng.
    *   LLM bước 4: Tổng hợp câu trả lời cuối cùng kèm theo nguồn (sources).

---

### 5. Đánh giá Tổng kết

**Điểm mạnh:**
*   **Context Preservation:** Giữ được dòng thời gian hoàn hảo, điều mà Vector DB thường làm mất.
*   **Resource Efficiency:** Cực kỳ tiết kiệm tài nguyên nhờ cơ chế chụp ảnh theo sự kiện.
*   **Privacy:** Chạy offline hoàn toàn được với Ollama/vLLM.

**Thách thức:**
*   **LLM Cost:** Việc tóm tắt tầng bậc (hierarchical summarization) tiêu tốn khá nhiều token nếu người dùng hoạt động liên tục. Tác giả đã giảm thiểu bằng cách cho phép cấu hình `mouse_cluster_gap`.
*   **Dependency:** Phụ thuộc nặng vào khả năng Vision của model để hiểu các cụm thao tác chuột (L0).

CatchMe là một ví dụ điển hình cho xu hướng **"Agentic Memory"** - nơi AI không chỉ tìm kiếm dữ liệu mà còn thực sự "suy luận" lại quá trình làm việc của con người thông qua cấu trúc dữ liệu phân cấp.