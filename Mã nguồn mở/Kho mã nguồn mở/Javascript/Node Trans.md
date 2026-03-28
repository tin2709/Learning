Dưới đây là bản phân tích chi tiết về dự án **Node Trans**, một ứng dụng dịch thuật và chuyển đổi giọng nói thành văn bản (STT) thời gian thực, hỗ trợ cả Cloud và Offline.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng một "Combo" công nghệ hiện đại, kết hợp giữa Web, Desktop và AI:

*   **Frontend:** **React 19** (phiên bản mới nhất) kết hợp với **Vite** và **Tailwind CSS v4**. Sử dụng **Socket.io-client** để giao tiếp thời gian thực.
*   **Backend:** **Node.js** với framework **Express 5**. Quản lý kết nối bền vững bằng **Socket.io**.
*   **Desktop App:** **Electron**, cho phép chạy ứng dụng trên macOS và Windows với khả năng can thiệp sâu vào hệ thống (như ghi âm hệ thống).
*   **Audio Processing:** **FFmpeg** là "trái tim" của việc xử lý âm thanh, sử dụng `avfoundation` (macOS) và `dshow` (Windows) để bắt luồng âm thanh từ mic và hệ thống.
*   **STT Engines:** 
    *   **Cloud:** Soniox API (ưu tiên tốc độ và độ chính xác cao).
    *   **Local (Offline):** `nodejs-whisper` (dựa trên **whisper.cpp**) và **openai-whisper** (Python).
*   **AI Offline (Python ecosystem):** 
    *   **Pyannote-audio 3.1:** Dùng để phân tách người nói (Speaker Diarization).
    *   **Torch/CUDA/MPS:** Tăng tốc phần cứng qua GPU (NVIDIA) hoặc Apple Silicon.
    *   **Ollama/LibreTranslate:** Xử lý dịch thuật Offline.
*   **Database:** **SQLite** (`better-sqlite3`) để lưu trữ lịch sử phiên làm việc cục bộ.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc dự án được thiết kế theo mô hình **Hybrid & Orchestrated**:

*   **Kiến trúc Đa luồng (Subprocess Orchestration):** Node.js đóng vai trò điều phối (Orchestrator). Nó không trực tiếp xử lý AI mà spawn (tạo) các tiến trình Python hoặc C++ (`whisper-cli`) và giao tiếp qua cơ chế **Stdin/Stdout**. Điều này giúp tách biệt logic ứng dụng và các tác vụ nặng về tính toán.
*   **Sliding Window Processing (Xử lý cửa sổ trượt):** Đối với Local Whisper, dự án áp dụng chiến thuật cửa sổ trượt 6 giây (với 4 giây bước nhảy và 2 giây chồng lấp). Tư duy này giúp Whisper có đủ ngữ cảnh để nhận diện chính xác mà vẫn đảm bảo độ trễ thấp cho kết quả "Partial" (kết quả tạm thời).
*   **Tách biệt Audio Capturing:** FFmpeg được tách thành một service riêng, chuẩn hóa âm thanh về định dạng **PCM s16le, 16kHz, mono**. Điều này đảm bảo dù nguồn âm thanh là gì (Mic hay Loopback), các Engine STT đều nhận được dữ liệu đầu vào đồng nhất.
*   **Lazy Loading (Tối ưu hóa khởi động):** Các module nặng (như database, AI engines) được import theo kiểu `lazy` (chỉ load khi cần), giúp ứng dụng khởi động tức thì trên Electron.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Bridge Protocol (Node-Python JSON-RPC):** Dự án tự xây dựng giao thức truyền nhận dữ liệu qua luồng stdin/stdout bằng định dạng **Newline-delimited JSON**. Base64 audio được đẩy vào Python, và Python trả về kết quả JSON theo thời gian thực.
*   **Hard-exit skip (Python optimization):** Trong `transcribe.py`, tác giả sử dụng `os._exit(0)` để bỏ qua quá trình dọn dẹp (teardown) của Python interpreter, tránh lỗi crash buffer trên Windows khi unload các DLL của CUDA.
*   **Chunk Normalization:** Sử dụng `ChunkTransform` (kế thừa từ `stream.Transform` của Node.js) để gom các gói dữ liệu âm thanh nhỏ lẻ thành các khối chuẩn 120ms (3,840 bytes). Kỹ thuật này cực kỳ quan trọng để đảm bảo Engine STT không bị quá tải bởi quá nhiều gói tin nhỏ.
*   **Native Module Handling:** Tự động hóa việc rebuild `better-sqlite3` giữa môi trường Node.js và Electron ABI thông qua các script trong `package.json`.
*   **Overlay & Multi-window Management:** Electron quản lý hai cửa sổ: Main Window và một Overlay Window (trong suốt, không khung, luôn hiện trên cùng). Dữ liệu được đồng bộ giữa hai cửa sổ qua IPC (Inter-Process Communication).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của dữ liệu từ âm thanh đến văn bản:

1.  **Khởi động:** Người dùng chọn nguồn âm thanh (Mic/System/Both) và Engine (Soniox/Local).
2.  **Capture:** 
    *   Backend gọi **FFmpeg**. 
    *   FFmpeg bắt âm thanh từ driver và stream dữ liệu PCM về Node.js.
3.  **Preprocessing:** 
    *   `capture.js` nhận dữ liệu, cắt thành từng chunk 120ms.
    *   Nếu người dùng tạm dừng (Pause), Node.js sẽ "Gate" (chặn) luồng dữ liệu này, không gửi tiếp cho AI nhưng vẫn giữ kết nối FFmpeg.
4.  **Inference (STT):**
    *   **Cloud:** Gửi chunk qua gRPC/WebSocket đến Soniox.
    *   **Local:** Đẩy base64 audio qua Stdin của `whisper-worker.py`. 
    *   Đồng thời, nếu bật Diarization, `diarize.py` sẽ phân tích đặc trưng giọng nói để gán nhãn `SPEAKER_00`, `SPEAKER_01`.
5.  **Translation:** Kết quả văn bản thô được gửi qua **Ollama** hoặc **LibreTranslate** để dịch sang ngôn ngữ mục tiêu.
6.  **Broadcast:** 
    *   Node.js nhận kết quả cuối cùng (Utterance), lưu vào **SQLite**.
    *   Đẩy kết quả qua **Socket.io** đến React UI và Overlay Window.
7.  **Display:** React render văn bản với màu sắc khác nhau cho từng Speaker và hiển thị trên cửa sổ Overlay mờ ảo.

---

### Tổng kết
**Node Trans** là một dự án có độ phức tạp kỹ thuật cao, xử lý tốt bài toán tích hợp giữa nhiều ngôn ngữ (JavaScript, Python, C++) và các luồng dữ liệu nhạy cảm về thời gian (Real-time stream). Đây là một kiến trúc mẫu cho các ứng dụng **Local-first AI** hiện nay.