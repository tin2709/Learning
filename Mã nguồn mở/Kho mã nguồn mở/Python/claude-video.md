Bản phân tích chi tiết về kho lưu trữ **`bradautomates/claude-video`** (Công cụ `/watch` dành cho Claude):

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Dự án này là một ví dụ điển hình về việc kết hợp các công cụ dòng lệnh (CLI) kinh điển với sức mạnh của mô hình ngôn ngữ lớn (LLM) đa phương thức.

*   **Xử lý Video/Audio:** 
    *   **`yt-dlp`**: Dùng để tải xuống và trích xuất Metadata/Captions từ hàng trăm nền tảng (YouTube, TikTok, X...).
    *   **`ffmpeg` & `ffprobe`**: Trái tim của quá trình xử lý kỹ thuật. Dùng để lấy thông tin video (probe), trích xuất khung hình (frame extraction) và xử lý âm thanh (mono 16kHz) cho Whisper.
*   **Trí tuệ Nhân tạo (AI Inference):**
    *   **Whisper (vvia Groq/OpenAI)**: Chuyển đổi âm thanh thành văn bản khi không có phụ đề gốc. Groq được ưu tiên nhờ tốc độ cực nhanh (LPU).
    *   **Claude Multimodal (Vision)**: Sử dụng công cụ `Read` để "nhìn" các khung hình JPEG được trích xuất.
*   **Ngôn ngữ lập trình:** **Python 3** nhưng với một triết lý cực đoan: **"Zero-dependency"**. Các script như `whisper.py` tự xây dựng `multipart/form-data` thủ công bằng `urllib` thay vì dùng thư viện `requests` hay SDK chính thức, giúp plugin cực kỳ nhẹ và dễ cài đặt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `/watch` giải quyết bài toán "Quá tải ngữ cảnh" (Context Window Bloat) một cách rất thông minh:

*   **Cơ chế "Ngân sách khung hình" (Frame Budgeting):** Thay vì trích xuất khung hình theo tốc độ cố định (ví dụ 1 fps), hệ thống sử dụng logic **Auto-scaled FPS**. Video càng dài, mật độ khung hình càng thưa dần (giới hạn cứng 100 frames) để đảm bảo không làm "tràn" token của Claude nhưng vẫn giữ được nội dung tổng quát.
*   **Chiến lược Phụ đề Đa tầng (Multi-tier Transcription):**
    1.  Ưu tiên 1: Phụ đề có sẵn (Native/Auto-generated) qua `yt-dlp` (Miễn phí, tức thì).
    2.  Ưu tiên 2: Whisper qua Groq (Rẻ, nhanh).
    3.  Ưu tiên 3: Whisper qua OpenAI (Tin cậy nhưng đắt hơn).
*   **Tư duy "Focused Mode":** Cho phép người dùng giới hạn dải thời gian (`--start`, `--end`). Khi phạm vi hẹp lại, mật độ khung hình tự động tăng lên (lên tới 2 fps) để quan sát chi tiết hơn.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Xử lý Rolling Duplicates trong VTT:** Một vấn đề kinh điển của phụ đề tự động trên YouTube là các dòng chữ xuất hiện kiểu "cuốn" (rolling), dẫn đến dữ liệu thô bị lặp lại hàng chục lần. File `transcribe.py` có hàm `_dedupe` để gộp các đoạn text trùng lặp liên tiếp, làm sạch dữ liệu trước khi đưa vào LLM.
*   **Giao thức Skill "Im lặng":** File `setup.py` được thiết kế để chạy `--check` trong <100ms trước mỗi lần gọi lệnh. Nếu mọi thứ ổn, nó không in gì ra cả. Điều này tối ưu trải nghiệm người dùng (UX), tránh rác output trong cửa sổ chat.
*   **Multipart/Form-data thủ công:** Trong `whisper.py`, việc sử dụng `uuid` để tạo `boundary` và `BytesIO` để đóng gói dữ liệu nhị phân là kỹ thuật lập trình hệ thống ở trình độ cao, giúp loại bỏ hoàn toàn sự phụ thuộc vào các thư viện bên thứ ba.
*   **Tương thích Windows:** Xử lý tinh tế sự khác biệt giữa lệnh `python3` (Unix) và `python` (Windows), cũng như xử lý lỗi encoding (cp1252) bằng cách loại bỏ emoji trong các log quan trọng.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Trigger:** Người dùng nhập `/watch <URL>`.
2.  **Preflight:** `setup.py` kiểm tra `ffmpeg`, `yt-dlp` và API Key.
3.  **Ingestion:** `download.py` gọi `yt-dlp`. Nếu là file cục bộ, nó bỏ qua bước tải.
4.  **Decomposition:** 
    *   `frames.py` tính toán FPS dựa trên thời lượng và trích xuất ảnh vào thư mục tạm.
    *   `transcribe.py` hoặc `whisper.py` tạo file văn bản có timestamp.
5.  **Integration:** Một báo cáo Markdown được tạo ra, liệt kê đường dẫn ảnh và nội dung thoại.
6.  **Perception:** Claude gọi công cụ `Read` trên danh sách các file ảnh đó (thực hiện song song).
7.  **Synthesis:** Claude tổng hợp thông tin từ hình ảnh (Vision) và văn bản (Transcript) để trả lời người dùng.
8.  **Cleanup:** In ra đường dẫn thư mục tạm để người dùng hoặc hệ thống xóa sau khi xong việc.

### Tổng kết
Đây là một dự án **Tool-use** mẫu mực. Nó không cố gắng làm mọi thứ trong một script khổng lồ mà chia nhỏ thành các module chức năng (download, frames, transcribe, whisper) được liên kết chặt chẽ bởi một "hợp đồng" (contract) định nghĩa trong `SKILL.md`. Điểm mạnh nhất của nó là khả năng tối ưu hóa chi phí token thông qua việc quản lý mật độ khung hình thông minh.