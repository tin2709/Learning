Dựa trên tệp nguồn khổng lồ của dự án **Screenity**, đây là phân tích chi tiết về kiến trúc và kỹ thuật của ứng dụng ghi màn hình mã nguồn mở mạnh mẽ nhất cho Chrome hiện nay.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án là một ứng dụng **Chrome Extension (Manifest V3)** phức tạp, kết hợp giữa xử lý media thời gian thực và giao diện người dùng hiện đại:

*   **Frontend Framework:** React 18.2 kết hợp với **Radix UI** (để xây dựng các thành phần như Popover, Dialog, Slider với độ truy cập cao).
*   **Xử lý đồ họa & Annotation:**
    *   **Fabric.js:** Thư viện chính để quản lý lớp vẽ (canvas) phía trên trang web, cho phép thêm mũi tên, văn bản, và hình khối.
    *   **MediaPipe (@mediapipe/tasks-vision):** Sử dụng AI để tách nền (Background Removal) và làm mờ nền (Blur) cho webcam trong thời gian thực.
*   **Media & Encoding:**
    *   **WebCodecs API:** Công nghệ mới nhất để mã hóa video MP4 trực tiếp trong trình duyệt với hiệu suất cao (thay thế cho MediaRecorder truyền thống).
    *   **FFmpeg.wasm:** Sử dụng trong trình biên tập (Editor) để xử lý video, cắt ghép, và chuyển đổi định dạng (WebM sang MP4/GIF).
    *   **Web Audio API:** Quản lý trộn âm thanh (System audio + Microphone) thông qua `AudioContext`.
*   **Lưu trữ & Persistence:**
    *   **IndexedDB (via localforage):** Lưu trữ các mảnh video (chunks) tạm thời để tránh mất dữ liệu khi trình duyệt crash.
    *   **Chrome Storage API:** Quản lý cấu hình người dùng và trạng thái đồng bộ giữa các script.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Screenity được thiết kế theo mô hình **Phân tán (Distributed)** để vượt qua các hạn chế của môi trường Extension:

*   **Shadow DOM Encapsulation:** Toàn bộ giao diện Toolbar và Popup khi chèn vào trang web của người dùng được bao bọc trong **Shadow DOM**. Điều này ngăn chặn CSS của trang web gốc làm hỏng giao diện của Extension và ngược lại.
*   **Message Bus (Hệ thống tin nhắn):** Sử dụng một `messageRouter.js` tập trung. Thay vì gọi hàm trực tiếp, các thành phần (Background, Content Script, Popup) giao tiếp qua các "Event" định danh, giúp mã nguồn lỏng lẻo (decoupled) và dễ bảo trì.
*   **Offscreen Documents (MV3):** Theo chuẩn Manifest V3, Service Worker không thể truy cập DOM hoặc API Media lâu dài. Screenity giải quyết bằng cách tạo một **Offscreen Document** ẩn để duy trì luồng ghi (Stream) và xử lý âm thanh liên tục.
*   **Kiến trúc "Chunk-based Storage":** Video không được lưu dưới dạng một file lớn trong RAM. Dữ liệu được chia thành các mảnh nhỏ và đẩy liên tục vào IndexedDB. Điều này đảm bảo: 1. Không tràn bộ nhớ (OOM). 2. Có thể khôi phục video nếu tab bị đóng đột ngột.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Fast Recorder Gate & Laddering (Mã hóa thích ứng):** Trong file `fastRecorderGate.ts`, dự án triển khai kỹ thuật "Probing". Nó thử nghiệm các cấu hình mã hóa từ cao xuống thấp (H.264 High Profile -> Main -> Baseline) cho đến khi tìm thấy cấu hình mà phần cứng người dùng hỗ trợ tốt nhất.
*   **Canvas Synchronization:** Để hỗ trợ vẽ lên màn hình, Screenity tạo một lớp Canvas trong suốt bao phủ toàn bộ Viewport. Kỹ thuật pan/zoom được xử lý bằng cách tính toán lại tọa độ tuyệt đối so với vị trí cuộn trang (window.scrollX/Y).
*   **Audio Mixing (Gain Nodes):** Sử dụng `GainNode` để điều chỉnh âm lượng riêng biệt cho Microphone và System Audio trước khi đưa vào luồng ghi chính (`MediaStreamAudioDestinationNode`).
*   **Stale Lock Management:** Trong `Background/index.js`, dự án có cơ chế tự động dọn dẹp "stale locks" (các cờ trạng thái bị kẹt khi Service Worker chết bất thường) để đảm bảo lần khởi động tiếp theo không bị lỗi.
*   **Diagnostic Logging:** Hệ thống `diagnosticLog.js` triển khai một **Ring Buffer** lưu trữ 5 phiên làm việc gần nhất với tối đa 100 sự kiện mỗi phiên. Điều này cực kỳ quan trọng để hỗ trợ người dùng khi có lỗi ghi hình mà không xâm phạm quyền riêng tư.

---

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Luồng Khởi tạo Ghi hình (The Start Flow)
1.  **Trigger:** Người dùng nhấn tổ hợp phím hoặc click Popup.
2.  **Permission Check:** Kiểm tra quyền Camera/Microphone/Desktop Capture.
3.  **Setup Phase:** 
    *   Background script khởi tạo một `StartFlowTrace` để theo dõi hiệu năng.
    *   Tạo **Recorder Tab** (pinned) hoặc **Offscreen Document**.
4.  **Stream Acquisition:** Gọi `getDisplayMedia`. Nếu ghi vùng (Region), Extension sẽ tính toán tọa độ cắt (`CropTarget`).
5.  **Handshake:** Gửi tin nhắn "ready-to-record" đến Content Script để hiển thị bộ đếm ngược (Countdown).
6.  **Action:** Khi bộ đếm kết thúc, luồng media bắt đầu được đẩy vào bộ mã hóa (WebCodecs hoặc MediaRecorder).

#### B. Luồng Annotations (Vẽ và chú thích)
1.  Người dùng bật "Drawing Mode".
2.  Content Script kích hoạt Fabric.js Canvas.
3.  Khi người dùng vẽ, các đối tượng được thêm vào mảng quản lý của Fabric.
4.  Hệ thống `History.jsx` quản lý Undo/Redo bằng cách lưu trữ các snapshot trạng thái JSON của Canvas.

#### C. Luồng Kết thúc và Xử lý (The Post-Stop Flow)
1.  **Stop Signal:** Dừng stream, Recorder flushes tất cả dữ liệu còn lại vào IndexedDB.
2.  **Handoff:** Mở tab `EditorWebCodecs.html` hoặc `Editor.html`.
3.  **Processing:** 
    *   Nếu máy mạnh: Sử dụng WebCodecs để đóng gói (muxing) thành file MP4 ngay lập tức.
    *   Nếu máy yếu: Sử dụng FFmpeg.wasm để xử lý từng frame.
4.  **Output:** Tạo Blob URL để người dùng tải về hoặc upload lên Google Drive qua OAuth2 tích hợp.

### Tóm tắt đặc điểm nổi bật
Screenity không chỉ là một ứng dụng ghi màn hình đơn giản; nó là một bài học về cách tối ưu hóa hiệu suất **Javascript** cho các tác vụ nặng (Media encoding) và cách xử lý giao diện phức tạp trong môi trường hạn chế của trình duyệt. Việc sử dụng **WebCodecs** và **MediaPipe** cho thấy dự án luôn cập nhật những công nghệ trình duyệt tiên tiến nhất.