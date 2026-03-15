Chào bạn, đây là bản phân tích chi tiết về dự án **WeFlow** dựa trên cấu trúc thư mục và mã nguồn bạn cung cấp. WeFlow là một ứng dụng Desktop mạnh mẽ, tập trung vào việc xử lý dữ liệu địa phương từ WeChat.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, kết hợp giữa hiệu năng xử lý dữ liệu lớn và giao diện người dùng mượt mà:

*   **Framework chính:** **Electron 39** (kết hợp với Vite). Cho phép xây dựng ứng dụng Desktop bằng web tech nhưng vẫn truy cập được sâu vào tài nguyên hệ thống.
*   **Giao diện (Frontend):** **React 19**, **TypeScript**, và **Zustand** để quản lý state. Ngôn ngữ giao diện chính là Tiếng Trung (zh-CN).
*   **Xử lý ngôn ngữ & AI:** 
    *   **sherpa-onnx-node:** Chuyển đổi giọng nói thành văn bản (Speech-to-Text) ngoại tuyến.
    *   **jieba-wasm:** Phân tách từ (word segmentation) để tạo bản đồ từ khóa (word cloud).
    *   **silk-wasm:** Giải mã định dạng âm thanh đặc chủng của WeChat (.silk).
*   **Xử lý dữ liệu & Native:**
    *   **Koffi:** Thư viện FFI (Foreign Function Interface) tốc độ cao để gọi các hàm từ file `.dll` (Windows) hoặc `.dylib` (macOS), đặc biệt là `wx_key.dll` để trích xuất khóa giải mã.
    *   **Better-sqlite3:** Thao tác với cơ sở dữ liệu SQLite (vốn là định dạng WeChat sử dụng).
*   **Hình ảnh & Video:** **Sharp** (xử lý ảnh) và **ffmpeg-static** (giải mã video/live photo).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của WeFlow tuân thủ mô hình **Hybrid Desktop Architecture**:

*   **Kiến trúc Đa luồng (Multi-Worker):** Đây là điểm sáng nhất. Các tác vụ nặng như giải mã cơ sở dữ liệu (`wcdbWorker`), tạo báo cáo năm (`annualReportWorker`), tìm kiếm hình ảnh (`imageSearchWorker`) và nhận dạng giọng nói (`transcribeWorker`) đều được đẩy xuống **Node.js Worker Threads**. Điều này giúp Main Thread của Electron không bị treo, đảm bảo UI luôn phản hồi nhanh.
*   **Tách biệt tầng Native (Decoupling):** Logic giải mã và đọc dữ liệu nhạy cảm được đóng gói trong các service riêng (`keyService`, `wcdbService`). Ứng dụng không can thiệp trực tiếp vào bộ nhớ WeChat mà thông qua các kỹ thuật hooking/memory scanning chuyên biệt.
*   **Local-First & Privacy:** Toàn bộ quá trình giải mã, lưu trữ cache và phân tích đều diễn ra 100% tại máy người dùng. Không có dữ liệu chat nào được gửi lên server cloud.
*   **Cung cấp Gateway API:** WeFlow tích hợp một HTTP Server nội bộ (`httpService.ts`) chạy ở port 5031, cho phép các ứng dụng bên thứ ba (như ChatLab) truy vấn dữ liệu theo chuẩn JSON/ChatLab format.

---

### 3. Các kỹ thuật chính (Key Technical Techniques)

*   **Memory Scanning (Quét bộ nhớ):** Kỹ thuật trích xuất khóa giải mã 64 ký tự của database WeChat bằng cách quét vùng nhớ (RW sections) của tiến trình `WeChat.exe` đang chạy.
*   **XOR/AES Image Decryption:** WeChat lưu trữ ảnh trong các file `.dat` bằng thuật toán XOR hoặc AES. WeFlow thực hiện dò tìm khóa XOR tự động dựa trên header file để hiển thị ảnh gốc.
*   **Zstandard Decompression:** Sử dụng `fzstd` để giải nén các nội dung tin nhắn bị nén trong database.
*   **Database Hooking:** Theo dõi thời gian thực sự thay đổi của các file cơ sở dữ liệu WeChat để cập nhật tin nhắn mới nhất lên giao diện mà không cần khởi động lại ứng dụng.
*   **Vòng đời báo cáo (Annual Report Pipeline):** Quy trình gồm: Quét tin nhắn -> Phân tách từ (Jieba) -> Thống kê tần suất (Tally) -> Render trực quan hóa (ECharts) -> Xuất ảnh/PDF.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Giai đoạn Khởi tạo:**
    *   Ứng dụng tự động tìm kiếm thư mục cài đặt WeChat (`dbPathService`).
    *   Sử dụng `keyService` để tìm tiến trình WeChat đang chạy và "mượn" khóa giải mã database.
2.  **Giai đoạn Kết nối:**
    *   Khởi chạy `wcdbWorker` để mở các file DB (.db) bằng khóa đã lấy.
    *   Xây dựng cache cho danh sách liên lạc và avatar để tối ưu tốc độ hiển thị.
3.  **Giai đoạn Tương tác:**
    *   **Xem chat:** UI gửi yêu cầu qua IPC -> Worker đọc DB -> Trả về JSON -> React render.
    *   **Multimedia:** Khi gặp ảnh/video, `imageDecryptService` sẽ giải mã file `.dat` thành ảnh bình thường trong thư mục tạm.
4.  **Giai đoạn Phân tích & Xuất dữ liệu:**
    *   Người dùng chọn "Tạo báo cáo" -> Một luồng Worker riêng biệt sẽ duyệt qua hàng triệu dòng tin nhắn, thực hiện tính toán thống kê và trả về kết quả cuối cùng.
    *   Người dùng có thể xuất ra HTML (đã bao gồm CSS inline để xem offline hoàn chỉnh).

### Kết luận
**WeFlow** là một dự án có độ phức tạp kỹ thuật cao, xử lý cực tốt vấn đề tương tác giữa JavaScript và mã máy (Native). Nó không chỉ là một công cụ sao lưu mà là một nền tảng phân tích dữ liệu cá nhân (Personal Data Analytics) mạnh mẽ và bảo mật.