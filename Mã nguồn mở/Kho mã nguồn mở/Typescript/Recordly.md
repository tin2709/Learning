Đây là bản phân tích kỹ thuật chi tiết về dự án **Recordly** dựa trên mã nguồn và tài liệu bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Recordly là một ứng dụng Desktop phức tạp kết hợp giữa tầng giao diện web và tầng xử lý hệ thống máy tính thấp (Low-level).

*   **Framework chính:** **Electron** (Main process điều phối hệ thống, Renderer process cho UI).
*   **Ngôn ngữ:** **TypeScript** (chiếm ~90%), kết hợp với **C++** (Windows Capture) và **Swift** (macOS ScreenCaptureKit).
*   **Rendering Engine:** **PixiJS (v8)**. Đây là lựa chọn then chốt. Thay vì dùng DOM hay Canvas API thuần, dự án dùng PixiJS để tận dụng WebGL/WebGPU, giúp xử lý mượt mà hàng chục layer (video, cursor, webcam, annotations) và render export với hiệu suất cao.
*   **Xử lý Video/Audio:** 
    *   **FFmpeg:** Sử dụng thông qua `ffmpeg-static` để muxing (trộn) audio/video và mã hóa các định dạng phức tạp.
    *   **Native Helpers:** Các binary riêng được build từ C++/Swift để gọi API trực tiếp từ OS (WGC trên Windows, ScreenCaptureKit trên Mac) nhằm giảm độ trễ và tăng chất lượng hình ảnh.
*   **AI/Transcription:** **Whisper.cpp**. Dự án tích hợp mô hình `ggml-small.bin` để tự động tạo phụ đề (auto-captions) ngay trên máy người dùng (Local AI).
*   **UI/UX:** React, Tailwind CSS, Shadcn/UI và Lucide/Phosphor Icons.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Recordly được thiết kế theo hướng **"Separation of Concerns" (Tách biệt trách nhiệm)** rất rõ rệt:

*   **Kiến trúc Đa cửa sổ (Multi-window Strategy):**
    *   `HUD Overlay`: Một cửa sổ trong suốt, nhỏ gọn, luôn nổi trên cùng (`alwaysOnTop`) để điều khiển khi đang quay.
    *   `Source Selector`: Cửa sổ riêng để chọn màn hình/ứng dụng.
    *   `Editor`: Cửa sổ nặng nhất, chứa timeline và preview engine.
*   **Hệ thống Extension (Plugin Architecture):** Recordly xây dựng một "Sandbox" cho extension. Extension không có quyền truy cập trực tiếp vào Node.js mà phải thông qua một **Host API** được cấp phép (permissions-gated). Điều này bảo vệ hệ thống khỏi các script độc hại và đảm bảo hiệu suất render không bị ảnh hưởng.
*   **Tư duy "Non-destructive Editing":** Khi quay xong, video gốc được giữ nguyên. Mọi thao tác zoom, cắt, thêm cursor effects đều được lưu dưới dạng metadata trong file `.recordly`. PixiJS sẽ "vẽ" lại các hiệu ứng này dựa trên timeline theo thời gian thực.
*   **Cơ chế Capture Độc lập (Telemetry):** Thay vì chỉ quay phim màn hình, Recordly ghi lại **dữ liệu tọa độ chuột (Telemetry)** riêng biệt với tần suất ~30Hz. Điều này cho phép sau khi quay xong, người dùng có thể thay đổi kích thước chuột, thêm hiệu ứng bóng đổ hoặc làm mượt chuyển động chuột (smoothing) mà không cần quay lại từ đầu.

---

### 3. Kỹ thuật Lập trình Chính (Key Programming Techniques)

*   **Native Interop (Giao tiếp mã máy):** 
    *   Sử dụng `spawn` để chạy các binary trợ giúp (native helpers). 
    *   Dùng `uiohook-napi` để lắng nghe sự kiện chuột/bàn phím ở mức toàn cục hệ thống (Global Hooks), ngay cả khi ứng dụng đang bị ẩn.
*   **Pipeline Xuất Video (Export Pipeline):** 
    *   Kỹ thuật **Frame-by-Frame Piping**: Renderer sẽ vẽ từng khung hình lên một canvas ẩn, sau đó chuyển đổi thành dữ liệu thô (Raw pixels/RGBA) và "pipe" trực tiếp vào stdin của FFmpeg. 
    *   Điều này tránh việc phải tạo các file tạm khổng lồ và cho phép kiểm soát bitrate/chất lượng đầu ra một cách chính xác.
*   **Xử lý Bất đồng bộ (IPC handling):** Hệ thống IPC được module hóa cực kỳ tốt. Thay vì viết tất cả trong `main.ts`, dự án chia nhỏ thành các handlers: `registerRecordingHandlers`, `registerExportHandlers`, `registerProjectHandlers`... giúp mã nguồn dễ bảo trì.
*   **Quản lý Tài nguyên (Memory Management):** Sử dụng PixiJS textures và dispose (giải phóng) tài nguyên thủ công để tránh tràn bộ nhớ (memory leak) khi xử lý các file video 4K dài.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

#### Bước 1: Khởi tạo và Quay (Recording)
1.  Người dùng chọn nguồn (Cửa sổ hoặc Toàn màn hình).
2.  Electron gọi Native Helper tương ứng (ví dụ: `ScreenCaptureKitRecorder.swift` trên Mac).
3.  **Ghi song song:** 
    *   Video thô được ghi vào file tạm.
    *   Audio (System + Mic) được ghi thành các track riêng.
    *   Cursor Telemetry được lưu vào file JSON (tọa độ x, y, trạng thái click).

#### Bước 2: Biên tập (Editing)
1.  Editor nạp video vào PixiJS.
2.  Hệ thống đồng bộ hóa Timeline: Khi người dùng kéo thanh trượt, PixiJS tính toán vị trí chuột, mức độ zoom tại thời điểm `t` và render lên màn hình.
3.  Cửa sổ xem trước (Preview) thực chất là một canvas WebGL chạy ở tốc độ 60fps.

#### Bước 3: Xuất bản (Export)
1.  Người dùng chọn định dạng (MP4/GIF).
2.  Một tiến trình render "offline" bắt đầu: Nó chạy qua từng millisecond của dự án, vẽ khung hình và đẩy vào encoder.
3.  FFmpeg thực hiện bước cuối cùng: Trộn video đã render với các track audio và tạo file thành phẩm.

---

### Tổng kết
Recordly không chỉ là một ứng dụng Electron thông thường mà là một **Video Engine** thu nhỏ. Sự kết hợp giữa **Native API** (để có hiệu suất quay cao nhất) và **Web Technologies** (để có trải nghiệm UI linh hoạt) là điểm sáng nhất trong kiến trúc của dự án này. Việc tách rời dữ liệu chuột (Telemetry) khỏi dữ liệu hình ảnh là kỹ thuật then chốt giúp nó đạt được các hiệu ứng "pro-grade" giống như các phần mềm trả phí (Screen Studio).