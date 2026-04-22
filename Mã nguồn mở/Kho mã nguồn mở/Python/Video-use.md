Dưới đây là phân tích chi tiết về dự án **video-use**, một công cụ chỉnh sửa video mã nguồn mở dựa trên sự điều khiển của LLM (Claude Code).

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án này không cố gắng xây dựng một trình chỉnh sửa video mới từ đầu, mà đóng vai trò là "bộ não" điều phối các công cụ mạnh mẽ có sẵn:

*   **LLM (Claude Code):** Đóng vai trò là kiến trúc sư và người ra quyết định. Nó đọc kịch bản, hiểu nội dung qua văn bản và đưa ra các quyết định cắt ghép (EDL).
*   **FFmpeg & FFprobe:** "Xương sống" cho mọi thao tác xử lý video. FFmpeg thực hiện cắt, ghép, chỉnh màu (grading), xử lý âm thanh, chèn phụ đề và render cuối cùng.
*   **ElevenLabs Scribe:** Công cụ chuyển đổi âm thanh thành văn bản (ASR) với độ chính xác cao ở cấp độ từ (word-level timestamps), nhận diện người nói (diarization) và phát hiện sự kiện âm thanh (cười, vỗ tay).
*   **Manim (3Blue1Brown's engine):** Công cụ tạo hoạt ảnh (animation) toán học/kỹ thuật thông qua code Python, được dùng để tạo các overlay giải thích.
*   **Pillow (PIL) & Librosa:** Xử lý hình ảnh (để tạo `timeline_view`) và phân tích âm thanh (để tạo dạng sóng - waveform).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `video-use` phản ánh một bước đi thông minh trong việc vượt qua rào cản về "Context Window" và khả năng xử lý hình ảnh của LLM:

*   **"Video as Text" (Video dưới dạng văn bản):** Thay vì bắt LLM xem hàng nghìn khung hình (tốn hàng triệu token và nhiễu), hệ thống chuyển đổi video thành một file `takes_packed.md` cực nhẹ (~12KB). LLM "đọc" video để hiểu cấu trúc và nội dung.
*   **On-demand Visuals (Hình ảnh theo yêu cầu):** Chỉ khi LLM cần kiểm tra một điểm cắt nhạy cảm hoặc so sánh các take quay hỏng, nó mới gọi tool `timeline_view` để tạo ra một ảnh PNG tổng hợp. Đây là tư duy "Lazy Loading" trong xử lý dữ liệu đa phương tiện.
*   **Hệ thống Sub-agent song song:** Khi cần tạo nhiều hoạt ảnh phức tạp (Manim/Remotion), LLM sẽ phân tách nhiệm vụ cho các Agent con chạy song song, giúp tối ưu thời gian thực tế (wall time).
*   **Hệ thống tự đánh giá (Self-Evaluation Loop):** Đây là tư duy "Closed-loop": Sau khi render, AI tự kiểm tra lại sản phẩm tại các điểm cắt (cut boundaries) để tìm lỗi (giật hình, nổ âm thanh) trước khi trình chiếu cho người dùng.

### 3. Các kỹ thuật chính (Key Techniques)

Dự án áp dụng nhiều kỹ thuật chuyên nghiệp trong sản xuất video hậu kỳ:

*   **Cắt dựa trên ranh giới từ (Word-boundary Precision):** Các điểm cắt được ép (snap) chính xác vào thời điểm kết thúc/bắt đầu của một từ dựa trên dữ liệu từ Scribe, đảm bảo không bị mất chữ.
*   **Xử lý âm thanh chống nhiễu (Audio Pop Prevention):** Áp dụng kỹ thuật `afade` 30ms tại mọi điểm cắt để triệt tiêu tiếng "pop" kỹ thuật số khi nối các đoạn video khác nhau.
*   **Loudness Normalization (Chuẩn hóa âm lượng):** Sử dụng tiêu chuẩn `-14 LUFS` (chuẩn của YouTube/TikTok/Spotify) giúp video sẵn sàng để đăng tải ngay lập tức mà không bị quá to hay quá nhỏ.
*   **Per-segment Filter Chain:** Thay vì render một lần toàn bộ, hệ thống xử lý màu sắc và hiệu ứng trên từng đoạn nhỏ (extract) sau đó mới nối lại (concat lossless), giúp tránh việc nén video hai lần (double-encoding).
*   **Phụ đề tự động kiểu "Bold-Uppercase":** Kỹ thuật chia phụ đề thành các cụm 2 từ, viết hoa toàn bộ, căn giữa - một phong cách rất thịnh hành trên mạng xã hội hiện nay (Alex Hormozi style).

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Luồng làm việc (Pipeline) diễn ra theo 7 bước chặt chẽ:

1.  **Inventory (Kiểm kê):** Quét toàn bộ thư mục, dùng `ffprobe` lấy siêu dữ liệu và `transcribe_batch` để chuyển toàn bộ source sang JSON.
2.  **Packing (Đóng gói):** Gộp các file JSON thành `takes_packed.md`, chia đoạn dựa trên khoảng lặng (silence >= 0.5s).
3.  **Strategy (Chiến lược):** LLM phân tích văn bản, đề xuất kế hoạch chỉnh sửa (Ví dụ: "Tôi sẽ lấy take 2 của đoạn Hook vì take 1 bị vấp"). Người dùng cần xác nhận (Confirm) trước khi tiếp tục.
4.  **EDL Creation (Lập danh sách cắt):** LLM tạo ra file `edl.json` chứa các mốc thời gian, hiệu ứng màu và yêu cầu overlay.
5.  **Parallel Rendering:**
    *   Các sub-agent tạo hoạt ảnh Manim.
    *   Helper script cắt nhỏ video, chỉnh màu và xử lý âm thanh từng đoạn.
6.  **Stitching (Nối):** Dùng FFmpeg nối các đoạn clip (lossless) và chèn overlay/phụ đề ở lớp trên cùng (Rule: phụ đề luôn nằm trên cùng).
7.  **Self-Eval & Persist:** AI dùng `timeline_view` kiểm tra lại các điểm cắt trên file `preview.mp4`. Nếu đạt yêu cầu, nó sẽ cập nhật lịch sử vào `project.md` để ghi nhớ cho phiên làm việc sau.

---
**Kết luận:** `video-use` là một ví dụ điển hình về việc sử dụng LLM như một **"Tổng đạo diễn" (Director)** hơn là một **"Người dựng phim" (Video Editor)** thủ công. Nó tận dụng tối đa sức mạnh lập luận của AI trên văn bản để điều khiển các công cụ xử lý video truyền thống.