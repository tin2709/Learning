Dựa trên các tệp mã nguồn của dự án **fireworks-tech-graph**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** 
    *   **Python:** Đóng vai trò là "trái tim" của hệ thống xử lý logic và tạo hình ảnh (`generate-from-template.py`).
    *   **Shell Script:** Dùng để bao bọc (wrapper) các lệnh, thực hiện kiểm tra (validation) và quản lý quy trình tự động.
*   **Đồ họa & Hiển thị:**
    *   **SVG (Scalable Vector Graphics):** Là định dạng trung gian và đầu ra chính. Hệ thống sử dụng XML thuần để vẽ các khối (shapes) và đường nối (arrows).
    *   **`librsvg` (rsvg-convert):** Công cụ dòng lệnh quan trọng dùng để chuyển đổi SVG sang PNG chất lượng cao (mặc định 1920px) mà không làm mất chi tiết hoặc gây nhiễu nén.
*   **Hệ sinh thái AI:**
    *   **Claude Code Skill:** Được thiết kế như một "Skill" cho Claude Code, cho phép LLM hiểu các quy tắc vẽ đồ họa kỹ thuật phức tạp thông qua chỉ dẫn trong `SKILL.md`.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế theo mô hình **Style-Driven Generator** (Trình tạo dựa trên phong cách):

*   **Tách biệt dữ liệu và giao diện (Decoupling):** Cấu trúc logic của sơ đồ (node, arrow, layer) được định nghĩa bằng JSON, trong khi các thuộc tính thẩm mỹ (màu sắc, font, stroke) được quản lý qua các **Style Profiles** (Style 1-7).
*   **Hệ thống phân loại sơ đồ (Taxonomy):** Kiến trúc phân loại rõ ràng 14 loại biểu đồ UML và các Pattern đặc thù của AI/Agent (như RAG, Multi-Agent). Điều này giúp AI chọn đúng Shape Vocabulary (vốn từ vựng hình khối) phù hợp.
*   **Kiến trúc tham chiếu (Modular Reference):** Sử dụng các file Markdown trong thư mục `references/` để lưu trữ "tri thức" về icon, màu sắc và quy tắc bố cục, giúp hệ thống dễ dàng mở rộng thêm phong cách mới mà không cần sửa code core.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Orthogonal Routing (Định tuyến vuông góc):** Trong file `generate-from-template.py`, hệ thống sử dụng thuật toán để tính toán đường đi của mũi tên theo hình chữ L hoặc Z. Điều này giúp sơ đồ trông chuyên nghiệp và tránh cắt ngang qua các node.
*   **Collision Detection (Phát hiện va chạm):** Một kỹ thuật hình học được triển khai để đảm bảo mũi tên không đi xuyên qua nội dung của các node. Nếu phát hiện va chạm, hệ thống sẽ tính toán lại waypoint (điểm trung gian).
*   **Jump-over Arcs (Cung nhảy):** Khi hai đường thẳng giao nhau, hệ thống tự động thêm một hình bán nguyệt nhỏ (arc) để ký hiệu đường này nằm trên đường kia, giải quyết vấn đề nhầm lẫn trong các sơ đồ luồng phức tạp.
*   **XML Escaping & Validation:** Xử lý nghiêm ngặt các ký tự đặc biệt trong văn bản để đảm bảo file SVG luôn đúng cú pháp XML. Scripts `validate-svg.sh` kiểm tra sự cân bằng của thẻ, trích dẫn thuộc tính và các tham chiếu marker.
*   **Python List Method:** Kỹ thuật xây dựng SVG bằng cách append từng dòng vào một list trong Python trước khi ghi file, giúp tránh các lỗi truncation (cắt cụt) và dễ dàng debug từng phần tử đồ họa.

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình từ mô tả của người dùng đến hình ảnh cuối cùng diễn ra qua 6 bước:

1.  **Tiếp nhận & Phân loại:** AI nhận mô tả (ví dụ: "Vẽ sơ đồ RAG"). Dựa trên `SKILL.md`, nó phân loại đây là `Architecture Diagram` và mặc định chọn `Style 1`.
2.  **Trích xuất cấu trúc:** AI xác định danh sách các thành phần (Query, Embedding, Vector DB...) và các mối quan hệ giữa chúng.
3.  **Lập kế hoạch bố cục:** AI tính toán tọa độ (x, y) trên lưới 8px, đảm bảo khoảng cách tối thiểu giữa các node là 80px và giữa các layer là 120px.
4.  **Tạo dữ liệu trung gian:** AI chuẩn bị một cấu trúc JSON chứa toàn bộ thông tin về các node (vị trí, loại hình khối) và mũi tên (điểm bắt đầu, điểm kết thúc, nhãn).
5.  **Thực thi Generator:** 
    *   Gọi `generate-from-template.py` để dựng file SVG.
    *   Script Python sẽ áp các Token màu sắc từ phong cách đã chọn và thực hiện thuật toán định tuyến mũi tên.
6.  **Hậu xử lý & Xuất bản:** 
    *   `validate-svg.sh` kiểm tra tính hợp lệ của file SVG.
    *   `rsvg-convert` tạo ra file PNG 1920px.
    *   Hệ thống phản hồi đường dẫn tệp SVG và PNG cho người dùng.

### Tổng kết
**fireworks-tech-graph** không chỉ là một công cụ vẽ hình, mà là một **hệ thống chuyển đổi ngôn ngữ sang hình học có nhận thức về ngữ cảnh kỹ thuật**. Nó ưu tiên tính nhất quán của biểu tượng (ví dụ: LLM luôn có biểu tượng tia chớp) và sự sạch sẽ của kỹ thuật (đường nối vuông góc, không va chạm), giúp tạo ra các sơ đồ đạt tiêu chuẩn xuất bản chỉ từ các câu lệnh đơn giản.