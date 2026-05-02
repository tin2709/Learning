Dựa trên tài liệu kỹ thuật và mã nguồn của dự án **OpenWhispr**, dưới đây là bản phân tích chuyên sâu về kiến trúc và công nghệ của hệ thống dictation (đọc - chép) mã nguồn mở này.

---

### 1. Công nghệ cốt lõi (Core Technologies)

OpenWhispr là một ứng dụng desktop phức hợp, kết hợp giữa công nghệ Web hiện đại và các thư viện tính toán hiệu năng cao (Native):

*   **Runtime & UI Stack:** Electron 41, React 19, TypeScript và Tailwind CSS v4. Việc sử dụng Electron cho phép ứng dụng can thiệp sâu vào hệ điều hành (phím tắt toàn cục, ghi âm hệ thống).
*   **Speech-to-Text (STT):** 
    *   **Local:** Sử dụng `whisper.cpp` (C++ port của OpenAI Whisper) và NVIDIA Parakeet (thông qua `sherpa-onnx`) để xử lý ngoại tuyến.
    *   **Cloud:** Tích hợp OpenAI Whisper API cho tốc độ cao.
*   **AI Reasoning (LLM):** Sử dụng `llama.cpp` cho các mô hình chạy cục bộ (GGUF) và hỗ trợ đa nền tảng Cloud (GPT-5, Claude 4.7, Gemini 3.1) thông qua kiến trúc Provider.
*   **Audio Engine:** FFmpeg (đóng gói sẵn qua `ffmpeg-static`) để chuyển đổi định dạng và tiền xử lý âm thanh.
*   **Dữ liệu & Search:** 
    *   **Metadata:** `better-sqlite3` quản lý lịch sử và ghi chú.
    *   **Semantic Search:** Qdrant (Vector Database) chạy dưới dạng sidecar và mô hình `all-MiniLM-L6-v2` (ONNX) để thực hiện tìm kiếm theo ngữ nghĩa.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OpenWhispr tập trung vào sự **Riêng tư (Privacy-first)** và **Tính ổn định (Resilience)**:

#### A. Tách biệt quy trình (Process Isolation)
Để tránh việc lỗi bộ nhớ khi chạy các mô hình AI nặng làm sập ứng dụng chính, dự án chia thành:
*   **Main Process:** Điều phối luồng, quản lý cửa sổ và IPC.
*   **Renderer Process:** Hiển thị giao diện người dùng.
*   **Utility Processes (Sidecars):** Các engine như `whisper-server`, `llama-server`, và `Qdrant` chạy như các tiến trình độc lập. Nếu một tiến trình AI bị crash, ứng dụng chính vẫn hoạt động.
*   **ONNX Worker:** Một "Utility Process" riêng biệt của Electron để chạy inference (nhúng văn bản, định danh người nói) nhằm cô lập hoàn toàn các lỗi native `bad_alloc`.

#### B. Thiết kế Đa cửa sổ (Dual Window Architecture)
*   **Main Window:** Một overlay nhỏ gọn, luôn nằm trên cùng (Always-on-top), hỗ trợ kéo thả để thực hiện việc dictation nhanh.
*   **Control Panel:** Giao diện quản lý đầy đủ cho cài đặt, lịch sử và cấu hình mô hình.

#### C. Chiến lược Hybrid (Cloud-Edge)
Hệ thống cho phép người dùng chuyển đổi linh hoạt giữa xử lý tại chỗ (an toàn dữ liệu) và xử lý đám mây (tốc độ cao) cho mọi tính năng cốt lõi.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

#### A. Can thiệp Native đa nền tảng (Native Hooks)
Dự án sử dụng mã nguồn C và Swift để xử lý các tính năng mà JavaScript không làm tốt:
*   **Windows:** Dùng C (`windows-key-listener.c`) thiết lập `SetWindowsHookEx` để bắt phím nóng cấp thấp (Push-to-Talk).
*   **macOS:** Dùng Swift (`macos-mic-listener.swift`) theo dõi thuộc tính CoreAudio để phát hiện khi nào mic được sử dụng mà không cần polling (tiết kiệm CPU).
*   **Linux:** Sử dụng D-Bus (`dbus-next`) để đăng ký phím tắt với GNOME/Hyprland trên Wayland (nơi mà phím tắt toàn cục thông thường bị chặn vì bảo mật).

#### B. Kỹ thuật "Auto-pasting" (Mô phỏng dán văn bản)
Để đưa văn bản vào con trỏ của bất kỳ ứng dụng nào, OpenWhispr triển khai các kỹ thuật riêng cho từng OS:
*   **macOS:** Sử dụng AppleScript thông qua quyền Accessibility.
*   **Windows:** Sử dụng PowerShell `SendKeys` hoặc `nircmd.exe`.
*   **Linux:** Sử dụng `XTest` (X11) hoặc `ydotool`/`wtype` (Wayland).

#### C. Tìm kiếm Hybrid (Reciprocal Rank Fusion - RRF)
Trong `vectorIndex.js`, ứng dụng kết hợp kết quả từ tìm kiếm văn bản truyền thống (FTS5 trong SQLite) và tìm kiếm Vector (Qdrant). Kỹ thuật RRF giúp gộp hai danh sách kết quả này lại để đưa ra những ghi chú liên quan nhất đến người dùng.

---

### 4. Luồng hoạt động hệ thống (System Workflows)

#### Luồng ghi âm và chuyển đổi (Dictation Flow):
1.  **Kích hoạt:** Người dùng nhấn Hotkey (Global Hook nhận diện).
2.  **Ghi âm:** `MediaRecorder API` ở Renderer thu thập dữ liệu âm thanh dưới dạng các đoạn (chunks).
3.  **Xử lý:** Khi dừng, Blob âm thanh được gửi qua IPC tới Main Process -> ghi vào file tạm -> gọi `whisper.cpp` hoặc Cloud API.
4.  **Hậu xử lý:** AI Agent thực hiện "cleanup" văn bản (sửa lỗi chính tả, ngắt câu).
5.  **Kết quả:** Văn bản được ghi vào Clipboard và tự động thực hiện lệnh "Paste" vào ứng dụng đang focus.

#### Luồng phát hiện cuộc họp (Meeting Detection):
1.  **Giám sát:** `MeetingProcessDetector` theo dõi các app như Zoom/Teams (macOS dùng workspace notifications, Win/Linux dùng polling danh sách tiến trình).
2.  **Định danh:** Nếu phát hiện âm thanh hệ thống (System Audio) đang chạy, ứng dụng kích hoạt chế độ ghi âm cuộc họp.
3.  **Diarization:** Âm thanh được gửi qua `diarization.js` để tách biệt các giọng nói khác nhau, gán nhãn người nói dựa trên "Voice Fingerprint".

### Tổng kết
OpenWhispr không chỉ là một app React-Electron thông thường mà là một hệ thống **Orchestration** (điều phối) phức tạp giữa các tiến trình native AI, cơ sở dữ liệu vector và các hàm hook sâu vào nhân hệ điều hành, tạo ra một trải nghiệm người dùng liền mạch tương tự như các phần mềm thương mại đóng gói (Granola, WisprFlow).