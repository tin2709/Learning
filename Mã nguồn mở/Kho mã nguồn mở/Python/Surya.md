Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **Surya**, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và kỹ thuật của hệ thống OCR thế hệ mới này.

### 1. Công nghệ cốt lõi (Core Tech Stack)

Surya không dựa trên các engine cũ như Tesseract mà sử dụng hoàn toàn các kiến trúc Deep Learning hiện đại:

*   **Framework:** Python 3.10+ kết hợp với **PyTorch** làm nền tảng tính toán chính.
*   **Kiến trúc Transformer:** Sử dụng thư viện **Hugging Face Transformers** để quản lý và triển khai các mô hình.
*   **Mô hình thị giác (Vision Models):**
    *   **EfficientViT:** Được tùy chỉnh cho bài toán phân tích phân đoạn ngữ nghĩa (semantic segmentation) để phát hiện vùng văn bản (Detection).
    *   **Donut (Document Understanding Transformer):** Một kiến trúc OCR không cần qua bước nhận diện ký tự cục bộ, xử lý trực tiếp từ ảnh sang văn bản (Recognition).
    *   **Segformer:** Sử dụng cho các tác vụ xử lý ảnh đầu vào.
*   **Mô hình ngôn ngữ (Language Models):** **DistilBERT** được sử dụng trong module `ocr_error` để nhận diện lỗi văn bản sau khi OCR.
*   **Hardware Optimization:** 
    *   **Flash Attention 2:** Tối ưu hóa tốc độ tính toán ma trận attention.
    *   **XLA (Accelerated Linear Algebra):** Hỗ trợ chạy trên Google TPU (thông qua `torch_xla`).
    *   **MPS:** Tối ưu cho chip Apple Silicon (M1/M2/M3).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Surya thể hiện tư duy **"Foundation Model"** (Mô hình nền tảng) kết hợp với **Modularity** (Tính module):

*   **Kiến trúc Hỗn hợp (Encoder-Decoder):** Hệ thống tách biệt rõ ràng giữa phần "Nhìn" (Vision Encoder - thường là Swin Transformer) và phần "Hiểu/Tạo" (Text Decoder). Điều này cho phép Surya thực hiện nhiều tác vụ như OCR, Layout, Table Rec chỉ với một cấu trúc khung duy nhất.
*   **Foundation Predictor:** Dự án đang chuyển dịch sang module `foundation/`, nơi một mô hình lớn có thể xử lý đa tác vụ (Multitasking) dựa trên các "Task Prompt" (ví dụ: `<OCR-WB>`, `<LAYOUT>`).
*   **Continuous Batching:** Kiến trúc `FoundationPredictor` được thiết kế để xử lý batch động, cho phép đưa các yêu cầu mới vào hàng đợi và xử lý song song, tối ưu hóa tối đa hiệu suất của GPU.
*   **Trừu tượng hóa thiết bị (Device Agnostic):** Mã nguồn xử lý rất kỹ việc tự động phát hiện và cấu hình Dtype (fp16, bf16, fp32) dựa trên phần cứng hiện có (CUDA, XLA, MPS, CPU).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Quản lý KV Cache thông minh:** Trong `foundation/cache/`, dự án triển khai hai loại Cache:
    *   `DynamicOpsCache`: Cho các thiết bị linh hoạt (GPU).
    *   `StaticOpsCache`: Cho XLA/TPU yêu cầu hình dạng (shape) cố định để tránh re-compilation. Kỹ thuật này bao gồm cả cơ chế **Sliding Window** để xử lý văn bản dài mà không làm nổ bộ nhớ.
*   **Xử lý hình học (Geometry Handling):** Lớp `PolygonBox` trong `common/polygon.py` xử lý các phép tính hình học phức tạp: Rescale theo DPI, tính diện tích giao nhau (IoU), chuyển đổi giữa Bbox và Polygon 4 góc.
*   **Tiling & Slicing:** Để xử lý các văn bản cực dài hoặc ảnh độ phân giải cao, Surya sử dụng kỹ thuật cắt ảnh (Slicing) thành các phần nhỏ (Chunks), xử lý độc lập rồi tổng hợp kết quả (Reconstruction).
*   **Custom Kernels & Compilation:** Sử dụng `torch.compile` với các chế độ tối ưu hóa khác nhau cho từng loại mô hình (Detection vs Recognition) để tăng tốc độ xử lý từ 1-11% tùy tác vụ.
*   **Validation dữ liệu:** Sử dụng **Pydantic** để định nghĩa schema cho kết quả đầu ra, đảm bảo tính nhất quán dữ liệu giữa model và giao diện người dùng.

### 4. Luồng hoạt động hệ thống (System Flow)

Quy trình xử lý một tài liệu trong Surya diễn ra như sau:

1.  **Input Stage:** Load PDF/Ảnh, chuyển đổi về không gian màu RGB. Nếu là PDF, sử dụng `pypdfium2` để render ảnh theo DPI được cấu hình (mặc định 96 cho detect, 192 cho OCR).
2.  **Detection Stage (Phát hiện):**
    *   Ảnh được đưa qua mô hình EfficientViT để tạo ra heatmap (bản đồ nhiệt).
    *   Hậu xử lý bằng OpenCV để tìm các đường bao (contours) và chuyển thành các Polygon chứa dòng văn bản.
3.  **Recognition Stage (Nhận dạng):**
    *   Các vùng ảnh (slices) chứa dòng văn bản được cắt ra và chuẩn hóa.
    *   Đưa vào mô hình Encoder-Decoder. Encoder trích xuất đặc trưng thị giác, Decoder giải mã autoregressive (tự hồi quy) để tạo ra văn bản và tọa độ ký tự/từ.
4.  **Enrichment Stage (Làm giàu dữ liệu):**
    *   **Layout Analysis:** Xác định vai trò của vùng (Header, Footer, Table, Figure).
    *   **Table Rec:** Nếu vùng là bảng, thực hiện nhận diện cấu trúc hàng/cột và ô (cells).
    *   **Reading Order:** Sắp xếp các khối văn bản theo thứ tự đọc logic của con người.
5.  **Post-processing Stage:**
    *   Làm sạch các tag LaTeX/Math.
    *   Xử lý văn bản lặp lại (Repetition penalty).
    *   Sửa lỗi chính tả bằng module OCR Error.
6.  **Output Stage:** Trả về kết quả dưới dạng JSON có cấu trúc hoặc hiển thị qua Streamlit GUI.

### Tổng kết
Surya là một ví dụ điển hình của việc ứng dụng **Generative AI vào thị giác máy tính**. Thay vì các quy trình xử lý ảnh truyền thống cứng nhắc, nó sử dụng khả năng "lập luận" của Transformer để hiểu cấu trúc tài liệu, giúp đạt được độ chính xác vượt trội trên 90+ ngôn ngữ và các tài liệu phức tạp như bài báo khoa học hay bảng biểu.