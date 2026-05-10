Chào bạn, đây là bản phân tích chuyên sâu về dự án **Meetily** - một hệ thống trợ lý cuộc họp AI ưu tiên quyền riêng tư, dựa trên dữ liệu mã nguồn bạn đã cung cấp.

---

### 1. Công nghệ Cốt lõi (Core Technologies)

Meetily là một minh chứng điển hình của việc kết hợp sức mạnh hệ thống (Rust) với sự linh hoạt của giao diện hiện đại (Next.js).

*   **Tauri Framework (v2):** Lớp vỏ bọc desktop thay thế Electron. Nó cho phép gọi các hàm Rust (Commands) từ JavaScript một cách an toàn và hiệu quả, giảm thiểu tài nguyên RAM.
*   **Audio Engine (Cung cấp bởi Rust):**
    *   **CPAL:** Thư viện xử lý âm thanh đa nền tảng, dùng để bắt luồng (stream) từ Microphone và System Audio.
    *   **Silero VAD (Voice Activity Detection):** Công nghệ lọc giọng nói, giúp hệ thống biết khi nào có người nói để gửi dữ liệu đi xử lý, tránh lãng phí tài nguyên khi im lặng.
    *   **RNNoise & EBU R128:** Kỹ thuật khử nhiễu và chuẩn hóa âm lượng theo tiêu chuẩn phát thanh chuyên nghiệp, giúp chất lượng âm thanh đầu vào cực kỳ ổn định.
*   **AI Transcription (STT):**
    *   **Whisper.cpp (Local):** Chạy mô hình Whisper của OpenAI ngay trên máy.
    *   **Parakeet (NVIDIA NeMo):** Sử dụng ONNX Runtime (`ort`) để đạt tốc độ xử lý gần như thời gian thực (Real-time).
*   **AI Summarization (LLM):**
    *   **Llama-helper (Sidecar):** Một chương trình Rust riêng biệt chạy song song, sử dụng `llama-cpp-2` để chạy các mô hình như Gemma 3 hoàn toàn offline.
    *   **API Connectors:** Hỗ trợ linh hoạt các nhà cung cấp cloud (Claude, Groq, OpenAI, OpenRouter).
*   **Data Persistence:**
    *   **SQLx (SQLite):** Lưu trữ dữ liệu quan hệ (cuộc họp, transcript) với hiệu năng cao.
    *   **IndexedDB:** Sử dụng ở phía Frontend như một lớp đệm "cứu hộ" (Recovery), đảm bảo không mất dữ liệu nếu ứng dụng bị crash đột ngột.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Meetily được thiết kế theo hướng **"Privacy-First"** và **"Resilient"** (Khả năng phục hồi cao).

*   **Modularization (Module hóa):** Toàn bộ hệ thống âm thanh được tách nhỏ thành các module chuyên biệt: `capture`, `devices`, `pipeline`, `saver`. Điều này giúp việc bảo trì và mở rộng (ví dụ thêm một nền tảng OS mới) trở nên dễ dàng.
*   **Pipeline Pattern:** Dữ liệu âm thanh đi qua một đường ống xử lý: `Raw Audio -> Resampling -> Enhancement -> Mixing -> VAD -> Transcription`. Cách tiếp cận này giúp kiểm soát chất lượng dữ liệu ở từng bước một cách độc lập.
*   **Sidecar Pattern:** Việc tách `llama-helper` thành một tiến trình riêng (Sidecar) là một quyết định thông minh. Nó giúp cô lập việc tiêu thụ tài nguyên cực lớn của LLM khỏi tiến trình chính của ứng dụng, tránh làm treo UI.
*   **Hybrid State Management:** Kết hợp trạng thái thời gian thực trong Rust (Atomic variables) và trạng thái giao diện trong React (Context API). Hệ thống đồng bộ hóa qua lại bằng các sự kiện (Events) của Tauri.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

Đội ngũ kỹ sư của Meetily đã áp dụng nhiều kỹ thuật tối ưu hóa hệ thống ở mức thấp:

*   **Adaptive Buffering (Bộ đệm thích ứng):** Hệ thống tự động nhận diện thiết bị là Wired hay Bluetooth. Với Bluetooth (vốn có độ trễ cao và jitter), hệ thống tự động tăng timeout của bộ đệm (80-200ms) để tránh hiện tượng vấp âm thanh.
*   **Persistent Resampler:** Thay vì tạo mới resampler cho mỗi đoạn âm thanh (gây mất năng lượng và sai số), hệ thống duy trì một resampler liên tục, giúp bảo toàn năng lượng âm thanh (RMS) chính xác đến 173.5% so với phương pháp thông thường.
*   **Professional Mixing với RMS Ducking:** Khi microphone có tiếng người nói, âm thanh hệ thống (ví dụ tiếng nhạc nền) sẽ tự động bị giảm âm lượng (ducking) để ưu tiên giọng nói, giúp bản ghi cực kỳ rõ ràng.
*   **Deduplication & Stop Words Cleaning:** Transcript được lọc bỏ các từ thừa (uh, um, ah) và xử lý lặp từ ngay ở mức hệ thống trước khi hiển thị lên giao diện.
*   **Zero-Overhead Logging:** Sử dụng các macro Rust (`perf_debug!`) để loại bỏ hoàn toàn các log gỡ lỗi trong bản build release, đảm bảo hiệu năng tối đa cho quá trình transcription vốn rất ngốn CPU.

---

### 4. Luồng Hoạt động Hệ thống (System Operation Flow)

Quy trình từ lúc nhấn nút "Record" đến khi có "Summary":

1.  **Giai đoạn Khởi tạo:**
    *   Frontend gọi `start_recording`.
    *   Rust kiểm tra quyền (Permission) và độ sẵn sàng của Model AI.
    *   Khởi tạo `IncrementalAudioSaver` để lưu các bản nháp âm thanh mỗi 30 giây (đề phòng crash).

2.  **Giai đoạn Xử lý Luồng (Hot Path):**
    *   Audio thu từ Mic và System được đưa vào `AudioMixerRingBuffer`.
    *   `ContinuousVadProcessor` theo dõi luồng âm thanh đã mix. Khi phát hiện một đoạn hội thoại (Speech Segment), nó được cắt ra và đưa vào hàng đợi xử lý.
    *   Các Worker transcription chạy song song xử lý hàng đợi này và bắn sự kiện `transcript-update` về UI.

3.  **Giai đoạn Kết thúc & Lưu trữ:**
    *   Người dùng nhấn "Stop". Luồng âm thanh dừng lại nhưng Pipeline vẫn tiếp tục chạy cho đến khi xử lý hết các đoạn âm thanh còn tồn đọng (Flush).
    *   Các file âm thanh (.checkpoints) được gộp lại (Merge) thành file `audio.mp4` cuối cùng bằng FFmpeg.
    *   Dữ liệu được lưu vào SQLite và xóa bản nháp trong IndexedDB.

4.  **Giai đoạn Tóm tắt (Summarization):**
    *   Frontend gửi toàn bộ Transcript đến `process_transcript`.
    *   Hệ thống kiểm tra độ dài (Token count). Nếu quá dài, nó sẽ chia nhỏ (Chunking), tóm tắt từng phần rồi gộp lại (Map-Reduce pattern).
    *   Kết quả cuối cùng được làm sạch (Markdown Cleaning) và hiển thị qua `BlockNote Editor`.

---

### Tổng kết
**Meetily** không đơn thuần là một ứng dụng wrapper cho AI. Nó là một hệ thống âm thanh phức tạp được tối ưu hóa sâu sắc cho phần cứng cục bộ, giải quyết các bài toán khó về độ trễ thiết bị không dây và tính toàn vẹn của dữ liệu trong môi trường offline. Đây là một mẫu hình lý tưởng cho các nhà phát triển muốn xây dựng ứng dụng desktop hiện đại bằng **Tauri + Rust**.