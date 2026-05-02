Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **VideoLingo**, đây là bản phân tích chi tiết về các khía cạnh công nghệ và kiến trúc của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

VideoLingo là một hệ thống "pipeline" xử lý video và ngôn ngữ phức hợp, kết hợp nhiều mô hình AI tiên tiến:

*   **Ngôn ngữ lập trình:** **Python 3.10** là ngôn ngữ chủ đạo.
*   **Giao diện người dùng:** **Streamlit**, giúp xây dựng ứng dụng web tương tác nhanh chóng cho các công cụ AI.
*   **Xử lý giọng nói (ASR):** **WhisperX**. Đây là lựa chọn chiến lược vì WhisperX hỗ trợ căn chỉnh ở mức độ từ (word-level alignment), giúp timestamps cực kỳ chính xác.
*   **Xử lý văn bản (NLP):** **spaCy** được dùng để phân tích cú pháp, chia nhỏ câu dựa trên cấu trúc ngữ pháp (ROOT, connectors).
*   **Mô hình ngôn ngữ lớn (LLM):** Hỗ trợ đa dạng từ GPT-4o, Claude 3.5 đến các mô hình nội địa như DeepSeek, thông qua giao thức API tương thích OpenAI.
*   **Xử lý âm thanh:**
    *   **Demucs:** Để tách lời bài hát và nhạc nền (vocal separation).
    *   **FFmpeg:** Công cụ "vạn năng" để chuyển đổi định dạng, gán phụ khẩu (burn-in) và trộn âm thanh.
*   **Chuyển đổi văn bản thành giọng nói (TTS):** Tích hợp rất nhiều backend từ đám mây (Azure, OpenAI) đến mã nguồn mở (GPT-SoVITS, F5-TTS, FishTTS).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế theo tư duy **Sequential Pipeline Architecture** (Kiến trúc đường ống tuần tự) với khả năng kiểm soát trạng thái:

*   **Phân rã theo mô-đun (Modularity):** Các bước xử lý được chia nhỏ thành các file đánh số từ `_1_` đến `_12_` trong thư mục `core/`. Mỗi file đảm nhận một nhiệm vụ duy nhất (Single Responsibility Principle), giúp dễ dàng debug và chạy riêng lẻ từng bước.
*   **Kiểm soát trạng thái (Persistence & Resumption):** Kết quả trung gian của mỗi bước (ASR, dịch, chia nhỏ câu) được lưu trữ dưới dạng file Excel (`.xlsx`) hoặc JSON trong thư mục `output/log/`. Điều này cho phép người dùng dừng và tiếp tục quá trình mà không phải làm lại từ đầu.
*   **Kiến trúc chuyển đổi (Translation Strategy):** Sử dụng quy trình **Translate-Reflect-Adapt** (Dịch - Phản hồi - Thích nghi). Thay vì dịch thẳng, LLM sẽ dịch thô, sau đó tự phản biện lỗi và cuối cùng là điều chỉnh cho mượt mà như văn phong của Netflix.
*   **Thích ứng tốc độ (Speed Adaptation):** Kiến trúc配音 (dubbing) tính toán "speed_factor" cho từng câu để đảm bảo âm thanh TTS khớp với thời lượng gốc của cảnh quay.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Decorator Pattern:** Sử dụng các decorator như `@except_handler` để quản lý lỗi/retry tự động và `@check_file_exists` để bỏ qua các bước đã hoàn thành, tối ưu hiệu suất.
*   **Xử lý đồng thời (Concurrency):** Sử dụng `ThreadPoolExecutor` trong quá trình gọi API dịch thuật và sinh âm thanh (TTS) để tăng tốc độ xử lý hàng trăm đoạn hội thoại cùng lúc.
*   **Cơ chế "Punctuation Restoration":** Đối với các ngôn ngữ như tiếng Trung, hệ thống sử dụng các mô hình Whisper đặc thù (như Belle-whisper) để khôi phục dấu câu chính xác, phục vụ cho việc chia câu bằng NLP.
*   **Kỹ thuật xử lý âm thanh luồng (Audio Engineering):** Tính toán độ lệch thời gian (tolerance) và chèn các đoạn tĩnh (silence) vào giữa các đoạn dubbing để tạo ra một bản track âm thanh hoàn chỉnh không bị lệch pha.
*   **JSON Repair:** Sử dụng thư viện `json_repair` để xử lý các phản hồi từ LLM không tuân thủ định dạng JSON nghiêm ngặt, tăng tính ổn định của hệ thống.

### 4. Luồng hoạt động hệ thống (System Workflows)

Quá trình dịch một video diễn ra qua 3 giai đoạn chính:

#### Giai đoạn 1: Trích xuất và Nhận diện (ASR Phase)
1.  Tải video (`yt-dlp`).
2.  Tách âm thanh gốc -> Tách giọng nói (Demucs).
3.  WhisperX nhận diện giọng nói và gán nhãn thời gian cho từng từ (Word-level timestamps).

#### Giai đoạn 2: Xử lý ngôn ngữ và Dịch thuật (NLP & Translation)
1.  **NLP Split:** spaCy chia nhỏ văn bản thô thành các câu logic.
2.  **Summarize:** LLM tóm tắt nội dung video và trích xuất bảng thuật ngữ (Terminology) để đảm bảo tính nhất quán (ví dụ: tên riêng không bị dịch sai).
3.  **3-Step Translation:** LLM dịch câu dựa trên ngữ cảnh xung quanh (câu trước/câu sau) và bảng thuật ngữ.
4.  **Sub-splitting:** Chia nhỏ các câu quá dài để đảm bảo chuẩn Netflix (chỉ hiển thị 1 dòng, không quá giới hạn ký tự).

#### Giai đoạn 3: Tổng hợp và Kết xuất (Synthesis Phase)
1.  **Dubbing Task:** Tạo danh sách các đoạn cần lồng tiếng.
2.  **TTS Generation:** Sinh âm thanh từ văn bản dịch. Nếu câu dịch dài hơn thời gian cho phép, LLM sẽ được gọi lại để rút gọn văn bản (`check_len_then_trim`).
3.  **Audio Merging:** Điều chỉnh tốc độ âm thanh -> Chèn khoảng lặng -> Trộn với nhạc nền gốc.
4.  **Final Render:** FFmpeg gộp video gốc + âm thanh lồng tiếng + phụ đề đã burn-in.

### Tổng kết
VideoLingo không chỉ là một công cụ dịch thuật, mà là một **AI Subtitle Team** hoàn chỉnh. Điểm mạnh nhất của nó nằm ở việc **kỹ thuật hóa (engineering)** các quy chuẩn thẩm mỹ ( Netflix-standard) vào trong thuật toán xử lý AI, thay vì chỉ dựa vào dịch thuật máy thông thường.