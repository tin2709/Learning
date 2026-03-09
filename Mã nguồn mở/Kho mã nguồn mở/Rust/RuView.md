Dựa trên toàn bộ mã nguồn và tài liệu bạn cung cấp, đây là bản phân tích chuyên sâu về dự án **RuView (WiFi-DensePose)**. Đây là một hệ thống cực kỳ phức tạp kết hợp giữa vật lý sóng vô tuyến (RF), xử lý tín hiệu số (DSP), kiến trúc phần mềm hiện đại và trí tuệ nhân tạo (AI) tiên tiến.

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology)

Công nghệ trọng tâm của RuView là **WiFi Sensing** dựa trên **Channel State Information (CSI)**.

*   **CSI vs RSSI:** Khác với RSSI (chỉ đo cường độ tín hiệu tổng thể), CSI cung cấp dữ liệu về biên độ (amplitude) và pha (phase) của từng **subcarrier** (sóng mang con) trong một gói tin WiFi (chuẩn OFDM). 
*   **Vật lý học:** Khi con người di chuyển hoặc hít thở, họ gây ra hiện tượng tán xạ, nhiễu xạ và phản xạ sóng WiFi. RuView phân tích sự thay đổi siêu nhỏ trong ma trận CSI để tái tạo lại hình dáng và hoạt động của vật thể.
*   **DensePose Mapping:** Dự án hiện thực hóa ý tưởng từ nghiên cứu của CMU, chuyển đổi dữ liệu RF không gian thấp thành bản đồ tọa độ UV trên bề mặt cơ thể người (DensePose), từ đó xác định 17 điểm chốt (keypoints) của bộ xương mà không cần camera.
*   **RuvSense (Multistatic Mesh):** Thay vì dùng 1 cặp thiết bị, RuView sử dụng một mạng lưới (mesh) các nút ESP32-S3, tạo ra hàng chục đường truyền chồng chéo, giúp tăng độ chính xác lên mức dưới 1 inch và nhìn xuyên tường hiệu quả hơn.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án này là một hình mẫu về kỹ thuật phần mềm hiện đại với các tư duy chủ đạo:

*   **Kiến trúc hướng miền (Domain-Driven Design - DDD):** Hệ thống được chia thành các "Bounded Contexts" rõ ràng (Sensing, Signal Processing, NN Inference, Triage cho cứu hộ...). Điều này giúp quản lý sự phức tạp khi mở rộng từ một dự án nghiên cứu thành một sản phẩm thương mại.
*   **ADR (Architecture Decision Records):** Với hơn 50 bản ghi ADR (từ 001 đến 053), mọi quyết định kỹ thuật (từ việc chọn thuật toán lọc nhiễu đến cấu trúc container RVF) đều được tài liệu hóa lý do, bối cảnh và hệ quả. Đây là tư duy "kiến trúc bền vững".
*   **Edge-First & No-Cloud:** Kiến trúc ưu tiên xử lý tại biên (trên ESP32 hoặc máy chủ cục bộ). Việc sử dụng **WASM (WebAssembly)** để chạy các module trí tuệ nhân tạo nhỏ ngay trên chip ESP32 là một bước đi đột phá, đảm bảo tính riêng tư tuyệt đối và độ trễ cực thấp.
*   **Rust-Centric (Performance):** Việc chuyển đổi từ Python sang Rust (v2) mang lại hiệu suất gấp **810 lần**, cho phép xử lý 54,000 khung hình/giây. Điều này cực kỳ quan trọng cho việc phân tích phổ tín hiệu thời gian thực.
*   **Hybrid Memory & Vector Search:** Sử dụng **HNSW (Hierarchical Navigable Small World)** từ thư viện RuVector để tìm kiếm vector đặc trưng tín hiệu, giúp nhận diện danh tính hoặc hoạt động gần như ngay lập tức.

---

### 3. Các kỹ thuật chính (Key Techniques & Algorithms)

RuView sử dụng một "kho vũ khí" toán học và AI đồ sộ:

*   **Xử lý tín hiệu (DSP):**
    *   **Hampel Filter:** Loại bỏ các giá trị ngoại lai (outliers) trong dữ liệu CSI thô.
    *   **SpotFi & Phase Unwrapping:** Hiệu chỉnh pha để loại bỏ sai số do phần cứng và sự lệch nhịp đồng hồ giữa các thiết bị.
    *   **Fresnel Zone Modeling:** Tính toán khoảng cách di chuyển dựa trên sự thay đổi pha trong các vùng Fresnel.
    *   **BVP (Body Velocity Profile):** Trích xuất vận tốc di chuyển của từng bộ phận cơ thể qua hiệu ứng Doppler.
*   **Trí tuệ nhân tạo tiên tiến (RuVector Ecosystem):**
    *   **SONA (Self-Optimizing Neural Architecture):** Tự học và điều chỉnh trọng số mô hình dựa trên môi trường thực tế mà không cần dữ liệu nhãn (Unsupervised Online Learning).
    *   **MERIDIAN (ADR-027):** Kỹ thuật giải quyết vấn đề "Domain Generalization". Nó giúp mô hình học cách tách biệt đặc trưng cơ thể người khỏi đặc trưng môi trường (tường, bàn ghế), giúp hệ thống hoạt động ở phòng mới mà không cần huấn luyện lại.
    *   **LoRA (Low-Rank Adaptation):** Cho phép tinh chỉnh mô hình cực nhanh trên phần cứng yếu bằng cách chỉ cập nhật một số lượng nhỏ trọng số.
    *   **Graph Neural Networks (GNN):** Sử dụng cấu trúc đồ thị để mô hình hóa mối quan hệ giữa các khớp xương người, đảm bảo tư thế tái tạo hợp lý về mặt vật lý.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Hệ thống hoạt động theo một chu trình khép kín từ phần cứng đến giao diện người dùng:

1.  **Capture (Thu thập):** Các nút **ESP32-S3** thu thập dữ liệu CSI thô từ sóng WiFi xung quanh (tần số ~20-100Hz).
2.  **Pre-processing (Tiền xử lý tại biên):** Áp dụng lọc Hampel, chuẩn hóa biên độ và hiệu chỉnh pha ngay trên chip hoặc tại Aggregator.
3.  **Feature Extraction (Trích xuất đặc trưng):** Chuyển đổi ma trận CSI thành các đặc trưng thời gian-tần số (Spectrogram, Doppler Profile).
4.  **Neural Inference (Suy luận):** 
    *   Mô hình AI (đóng gói trong định dạng `.rvf`) nhận dữ liệu đặc trưng.
    *   Sử dụng cơ chế Attention để tập trung vào các Subcarriers quan trọng nhất.
    *   Dự đoán tọa độ 17 điểm chốt cơ thể (Keypoints) và các chỉ số sinh tồn (BPM).
5.  **Adaptive Learning (Học thích nghi):** Hệ thống so sánh kết quả với mô hình môi trường (Field Model). Nếu có sai lệch ổn định, thuật toán SONA sẽ cập nhật nhẹ các trọng số (LoRA) để "quen" với căn phòng mới.
6.  **Visualization (Hiển thị):** Dữ liệu được đẩy qua WebSocket đến giao diện **Observatory (Three.js)** để hiển thị mô hình người 3D, bản đồ nhiệt tín hiệu và biểu đồ nhịp tim/hơi thở trong thời gian thực.

### Kết luận
RuView không chỉ là một dự án phần mềm, mà là một **hệ sinh thái trí tuệ nhân tạo tín hiệu (Signal Intelligence)**. Nó biến những thứ vô hình (sóng WiFi) thành dữ liệu thị giác hữu hình, mở ra kỷ nguyên mới cho giám sát y tế, an ninh và cứu hộ mà không xâm phạm quyền riêng tư như camera truyền thống.