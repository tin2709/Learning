Dưới đây là bản phân tích kỹ thuật chi tiết về **Cherry Studio - AI Core** (gói `packages/aiCore`) dựa trên tài liệu kiến trúc và mã nguồn bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology)

Hệ thống AI Core của Cherry Studio được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào tính mô-đun và hiệu suất:

*   **Vercel AI SDK (Lõi trung tâm):** Đây là "xương sống" của dự án. Thay vì tự viết logic gọi API cho từng nhà cung cấp (OpenAI, Anthropic, Google, v.v.), Cherry Studio tận dụng giao diện chuẩn hóa của Vercel AI SDK để xử lý streaming, gọi công cụ (tool calling) và xử lý lỗi.
*   **TypeScript (Strict Mode):** Toàn bộ hệ thống sử dụng TypeScript để đảm bảo an toàn về kiểu (type safety), đặc biệt là trong việc ánh xạ các tùy chọn cấu hình (Options) khác nhau của từng Provider.
*   **Dynamic Imports (Nhập động):** Các Provider (như `@ai-sdk/openai`, `@ai-sdk/anthropic`) được tải theo kiểu "lazy-loading". Điều này giúp giảm đáng kể dung lượng gói (bundle size) vì chỉ những gì được sử dụng mới được nạp vào bộ nhớ.
*   **Shadow DOM (Dành cho Renderer):** Sử dụng để hiển thị kết quả Markdown và SVG (như Mermaid, Graphviz) nhằm cô lập CSS, tránh xung đột với giao diện chính của ứng dụng.
*   **Pyodide:** Công nghệ chạy Python trực tiếp trong trình duyệt thông qua WebAssembly, cho phép thực thi code Python trong các khối mã mà không cần backend.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của AI Core tuân thủ các nguyên tắc thiết kế phần mềm sạch (Clean Architecture):

*   **Tách biệt mối quan tâm (Separation of Concerns):**
    *   **Layer Models (Tầng Mô hình):** Chỉ chịu trách nhiệm khởi tạo và cấu hình các đối tượng model.
    *   **Layer Runtime (Tầng Thực thi):** Chịu trách nhiệm thực hiện các yêu cầu (Text, Object, Image) và quản lý vòng đời của một request.
*   **Thiết kế hướng chức năng (Functional First):** Ưu tiên sử dụng các Factory Function (hàm nhà máy) thay vì phân cấp lớp (class) phức tạp. Ví dụ: `createModel()` là điểm vào duy nhất để tạo model, giúp mã nguồn dễ kiểm thử (testable).
*   **Kiến trúc "Cắm và Chạy" (Plug-and-Play):** Hệ thống được thiết kế để dễ dàng tích hợp thêm các SDK tương lai (như OpenAI Agents SDK) mà không phải đập đi xây lại phần lõi.
*   **Sự tối giản (Minimal Wrapping):** Triết lý thiết kế là "Bao bọc ít nhất có thể". AI Core không cố gắng che giấu Vercel AI SDK mà chỉ cung cấp thêm các lớp tiện ích (utility) và quản lý cấu hình xung quanh nó.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Plugin Engine (Động cơ Plugin):** Đây là điểm sáng kỹ thuật. Plugin Engine cho phép can thiệp vào toàn bộ vòng đời của một yêu cầu AI:
    *   *Pre-request:* Chỉnh sửa tham số trước khi gửi đi.
    *   *Stream Transformation:* Chỉnh sửa dữ liệu khi nó đang được đổ về (ví dụ: trích xuất thông tin tìm kiếm web hoặc lọc các thẻ suy nghĩ `<thought>`).
*   **Middleware System (Hệ thống Trung gian):** Tận dụng chuẩn Middleware của Vercel AI SDK để thực hiện các nhiệm vụ xuyên suốt như: Ghi log (Logging), xử lý lỗi (Error Handling), và điều chỉnh tham số đặc thù cho từng model (ví dụ: xử lý `reasoning_effort` của OpenAI o1).
*   **Provider Registry (Đăng ký Nhà cung cấp):** Một hệ thống quản lý tập trung các Provider. Nó cho phép ứng dụng tự động nhận diện và cấu hình các endpoint tương thích với OpenAI (như DeepSeek, Groq) một cách linh hoạt.
*   **Throttling & Debouncing (Tiết lưu và Chống rung):** Áp dụng trong việc cập nhật UI và lưu trữ DB khi nhận dữ liệu streaming. Kỹ thuật này giúp ứng dụng mượt mà, không bị treo khi AI phản hồi quá nhanh.
*   **Unified Error Mapping:** Chuyển đổi các lỗi thô từ API của các Provider khác nhau thành một hệ thống mã lỗi nội bộ đồng nhất, giúp UI hiển thị thông báo thân thiện với người dùng.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng xử lý một yêu cầu chat diễn ra như sau:

1.  **Khởi tạo:** Người dùng gửi tin nhắn. `useMessageOperations` (Hook) được gọi, kích hoạt một Redux Thunk (`sendMessage`).
2.  **Chuẩn bị Model:** `ModelFactory` nhận cấu hình từ database, gọi `createModel`. Tại đây, các `Middleware` chuẩn và `Provider` tương ứng được nạp.
3.  **Tiền xử lý (Plugin Engine):** Các Plugin (như Web Search) kiểm tra xem có cần kích hoạt công cụ bổ sung không.
4.  **Thực thi (Runtime):** `RuntimeExecutor` gọi hàm `streamText` của Vercel AI SDK.
5.  **Xử lý luồng (Streaming):**
    *   Dữ liệu đổ về theo từng đoạn (chunks).
    *   `StreamProcessor` phân loại dữ liệu: nội dung chính, suy nghĩ (thinking), hoặc gọi công cụ (tool call).
    *   Các Plugin thực hiện "Stream Transform" để làm sạch hoặc biến đổi dữ liệu nếu cần.
6.  **Cập nhật trạng thái:** Dữ liệu được đưa vào Redux theo cơ chế "Throttled" (tiết lưu) để cập nhật UI đồng thời lưu xuống IndexedDB để duy trì lịch sử.
7.  **Hoàn tất:** Khi stream kết thúc, các khối dữ liệu cuối cùng (như trích dẫn Citations hoặc hình ảnh) được xử lý và lưu trữ trạng thái `SUCCESS`.

**Kết luận:** AI Core của Cherry Studio không chỉ đơn thuần là một trình bao bọc API, mà là một **Runtime Environment** hoàn chỉnh cho AI, cho phép quản lý phức tạp các luồng dữ liệu suy nghĩ và công cụ một cách có hệ thống.