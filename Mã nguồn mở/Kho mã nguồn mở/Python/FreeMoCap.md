Dựa trên các tệp tin mã nguồn và tài liệu của dự án **FreeMoCap**, dưới đây là bản phân tích chuyên sâu về công nghệ cốt lõi, kiến trúc hệ thống, kỹ thuật lập trình và luồng hoạt động của hệ thống.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

FreeMoCap là một hệ sinh thái mã nguồn mở phức tạp, tận dụng sức mạnh của cộng đồng Python để xử lý thị giác máy tính và hình học không gian.

*   **Ngôn ngữ chủ đạo:** Python (3.10 - 3.12).
*   **Thị giác máy tính (CV) & Tracking:**
    *   **Mediapipe:** Engine chính để nhận diện 2D keypoints (Body, Hands, Face).
    *   **SkellyTracker:** Một wrapper trừu tượng cho phép chuyển đổi giữa các tracker (Mediapipe, YOLO, OpenPose).
    *   **OpenCV:** Xử lý luồng video, vẽ overlay và xử lý ảnh cơ bản.
*   **Tính toán toán học & Hình học:**
    *   **NumPy:** Xử lý mảng dữ liệu tọa độ đa chiều (số lượng camera, khung hình, số điểm, trục XYZ).
    *   **Anipose (customized):** Thư viện cốt lõi cho việc hiệu chuẩn camera (Calibration) và tái tạo 3D từ các điểm 2D (Triangulation).
*   **Giao diện người dùng (GUI):**
    *   **PySide6 (Qt for Python):** Xây dựng giao diện desktop đa nền tảng.
    *   **PyQtGraph:** Hiển thị cây thông số (Parameter Tree) và biểu đồ thời gian thực.
*   **Quản lý dữ liệu:**
    *   **Pydantic:** Định nghĩa các Data Models nghiêm ngặt, đảm bảo dữ liệu cấu hình và metadata luôn hợp lệ.
    *   **TOML/JSON:** Lưu trữ thông tin hiệu chuẩn camera và trạng thái GUI.
*   **Hệ sinh thái tích hợp:**
    *   **Blender:** Sử dụng thông qua một Addon riêng để render 3D và hoạt ảnh hóa skeleton.
    *   **Jupyter Notebook:** Tự động tạo báo cáo phân tích dữ liệu động.

### 2. Tư duy Kiến trúc (Architectural Logic)

Dự án đi theo mô hình **Modular Pipeline Architecture** (Kiến trúc đường ống mô-đun hóa):

*   **Tách biệt logic (Separation of Concerns):**
    *   `core_processes/`: Chứa logic nghiệp vụ thuần túy (tính COM, lọc dữ liệu, hiệu chuẩn). Không phụ thuộc vào giao diện.
    *   `data_layer/`: Lớp trừu tượng hóa việc đọc/ghi file. Sử dụng Pydantic để validate cấu trúc thư mục ghi hình.
    *   `gui/`: Lớp hiển thị, quản lý trạng thái tương tác của người dùng.
    *   `system/`: Quản lý các tài nguyên cấp hệ thống như Logging, Path getters, và User Data.
*   **Cơ chế Plugin cho Trackers:** Kiến trúc cho phép FreeMoCap không bị bó buộc vào Mediapipe. Thông qua `ModelInfo`, hệ thống có thể hiểu cấu trúc xương của bất kỳ tracker nào (như YOLO) chỉ cần định nghĩa lại danh sách landmark và kết nối xương.
*   **Kiến trúc Đa tiến trình (Multiprocessing):** Do Python có Global Interpreter Lock (GIL), dự án sử dụng `multiprocessing` để chạy các tác vụ nặng (như image tracking trên nhiều camera) nhằm tận dụng tối đa CPU đa nhân.

### 3. Kỹ thuật Lập trình Đặc sắc (Programming Techniques)

*   **Thread/Process Offloading:** Để GUI không bị "đơ" khi xử lý video, FreeMoCap sử dụng `QThread` (Worker) phối hợp với `multiprocessing.Process`. Kết quả trả về qua `multiprocessing.Queue` và được map ngược lại tín hiệu `Signal` của Qt để cập nhật giao diện.
*   **Strict Data Validation:** Sử dụng Pydantic v2 mạnh mẽ (ví dụ: `RecordingInfoModel`). Việc truy cập file không dựa trên hard-coded string mà qua các thuộc tính động, giúp giảm thiểu lỗi "file not found".
*   **Trừu tượng hóa hình học:** Các phép toán như xoay 90 độ trục X (`rotate_by_90_degrees_around_x_axis`) hay chiếu lên mặt phẳng Z được tách thành các hàm utility riêng biệt, có unit test đi kèm (`tests/test_geometry_utilities.py`).
*   **Cơ chế Logging tập trung:** Sử dụng một custom `LoggerBuilder` kết hợp với `DeltaTimeFilter` để đo thời gian thực thi giữa các log, cực kỳ hữu ích cho việc debug hiệu năng trong Computer Vision.
*   **Xử lý lỗi hệ thống:** Sử dụng `KillEventException` để dừng đột ngột các tiến trình tính toán nặng khi người dùng nhấn "Stop" mà không làm treo ứng dụng.

### 4. Luồng Hoạt động Hệ thống (System Data Flow)

Luồng dữ liệu đi qua các giai đoạn (Stages) cực kỳ nghiêm ngặt:

1.  **Stage 1: Capture/Import:**
    *   Dữ liệu video từ nhiều camera (USB/Webcam) được ghi đồng bộ hoặc người dùng import video có sẵn.
    *   Tạo cấu trúc thư mục: `synchronized_videos/` và `output_data/`.
2.  **Stage 2: Calibration (Hiệu chuẩn):**
    *   Sử dụng bảng Charuco. Hệ thống nhận diện các góc của bảng trên từng camera.
    *   **Logic mới (v1.7.0+):** Sử dụng vị trí bảng làm gốc tọa độ (Ground Plane) thay vì dùng vị trí camera 0, giúp dữ liệu ổn định hơn.
3.  **Stage 3: 2D Tracking:**
    *   Chạy Tracker (ví dụ Mediapipe) trên từng frame của từng camera.
    *   Kết quả là file `2dData_..._pixelXY.npy`.
4.  **Stage 4: Triangulation (Tái tạo 3D):**
    *   Kết hợp dữ liệu 2D từ tất cả camera và dữ liệu hiệu chuẩn camera.
    *   **Kỹ thuật nâng cao (v1.8.0):** **Outlier Rejection** – Loại bỏ các camera có sai số tái chiếu (reprojection error) cao để đảm bảo skeleton 3D không bị "giật" (ghost skeletons).
5.  **Stage 5: Post-processing:**
    *   **Interpolation:** Điền vào các khung hình bị mất điểm.
    *   **Butterworth Filter:** Lọc nhiễu tần số cao (làm mượt chuyển động).
    *   **Rigid Bone Enforcement:** Ép độ dài xương không đổi giữa các frame (dựa trên trung vị).
    *   **COM Calculation:** Tính toán trọng tâm cơ thể dựa trên các chỉ số nhân trắc học.
6.  **Stage 6: Visualization/Export:**
    *   Hiển thị qua `SkellyViewer`.
    *   Export sang `.blend` cho Blender hoặc `.ipynb` cho phân tích khoa học.

### Tổng kết
FreeMoCap không đơn thuần là một tool AI, mà là một **middleware** mạnh mẽ kết nối giữa phần cứng camera giá rẻ và các phần mềm phân tích chuyển động cao cấp. Sự kết hợp giữa **Qt (GUI)**, **Pydantic (Data Integrity)** và **Anipose (3D Geometry)** tạo nên một hệ thống nghiên cứu chuyển động chuẩn xác với chi phí cực thấp.