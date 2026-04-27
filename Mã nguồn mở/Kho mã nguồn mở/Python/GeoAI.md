Dựa trên mã nguồn của dự án **GeoAI**, đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và kỹ thuật:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này là một hệ sinh thái Python hiện đại kết hợp giữa Deep Learning (AI) và GIS (Hệ thống thông tin địa lý):

*   **Deep Learning Frameworks:** 
    *   **PyTorch:** Nền tảng chính cho huấn luyện và suy luận.
    *   **Hugging Face (Transformers & Hub):** Sử dụng cho các mô hình nền tảng (Foundation Models) như SAM (Segment Anything), DINOv2/v3, và Moondream (VLM).
    *   **Segmentation Models PyTorch (SMP):** Cung cấp các kiến trúc chuẩn như U-Net, DeepLabV3+.
    *   **timm (PyTorch Image Models):** Thư viện cho các kiến trúc phân loại hình ảnh.
*   **Geospatial Data Handling:**
    *   **Rasterio (GDAL):** Xử lý dữ liệu ảnh vệ tinh (GeoTIFF) và các phép toán ma trận địa lý.
    *   **GeoPandas & Shapely:** Quản lý dữ liệu vector (GeoJSON, Shapefile) và các phép toán hình học.
    *   **PySTAC:** Truy cập và tìm kiếm dữ liệu qua giao thức STAC (Spatial Temporal Asset Catalog).
*   **Visualization & UI:**
    *   **Leafmap & MapLibre:** Thư viện bản đồ tương tác trên môi trường Jupyter/Web.
    *   **PyQt (trong QGIS Plugin):** Xây dựng giao diện cho phần mềm QGIS.
*   **Inference Optimization:**
    *   **ONNX Runtime:** Hỗ trợ chạy mô hình ở định dạng ONNX để tối ưu hiệu suất và khả năng tương thích phần cứng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

GeoAI được thiết kế theo hướng **"Unified & Extensible"** (Thống nhất và Có thể mở rộng):

*   **Kiến trúc Phân lớp (Layered Architecture):**
    *   **Core Layer (`geoai/`):** Chứa các logic nghiệp vụ thuần túy về xử lý ảnh và mô hình AI.
    *   **CLI Layer (`geoai/cli.py` & `agent-harness/`):** Cung cấp giao diện dòng lệnh cho người dùng và các Agent AI (như Claude Code).
    *   **UI Layer (`qgis_plugin/`):** Tích hợp vào phần mềm QGIS giúp người dùng cuối không cần code vẫn sử dụng được AI.
*   **Kiến trúc Project-Oriented:** Sử dụng tệp JSON để quản lý trạng thái dự án (danh sách tệp, mô hình, kết quả), cho phép quy trình làm việc có tính kế thừa và có khả năng `undo/redo`.
*   **Thiết kế Hướng Module (Modular Design):** Tách biệt các chức năng thành các module chuyên biệt: `segment`, `detect`, `classify`, `change_detection`. Mỗi module có thể hoạt động độc lập hoặc phối hợp trong một `pipeline`.
*   **Agent-Friendly:** Điểm đặc biệt của dự án này là việc xây dựng `agent-harness` và `SKILL.md`. Đây là kiến trúc được tối ưu để các công cụ AI (Agent) có thể hiểu và gọi các lệnh địa không gian một cách chính xác.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Lazy Loading (PEP 562):** Trong `geoai/__init__.py`, hệ thống sử dụng kỹ thuật nạp chậm để tránh việc import các thư viện nặng (như torch hay tensorflow) khi chưa thực sự cần thiết, giúp tăng tốc độ khởi động CLI.
*   **Tiled/Windowed Inference:** Xử lý ảnh vệ tinh cực lớn bằng cách chia nhỏ thành các "chips" (ảnh nhỏ), chạy suy luận từng phần và ghép lại. Dự án sử dụng kỹ thuật **Spline Blending** để loại bỏ các đường viền (artifacts) tại điểm ghép nối giữa các ảnh nhỏ.
*   **Automatic Device Management:** Tự động phát hiện và cấu hình thiết bị phần cứng (CUDA cho NVIDIA, MPS cho Mac Silicon, hoặc CPU) mà không cần người dùng can thiệp thủ công.
*   **Data Augmentation tích hợp:** Sử dụng thư viện `albumentations` được tinh chỉnh cho dữ liệu địa lý (ví dụ: xoay ảnh nhưng vẫn giữ nguyên tọa độ địa lý).
*   **Asynchronous Subprocess:** Trong QGIS plugin, các tác vụ AI nặng được đẩy xuống các tiến trình con (subprocess) để tránh làm treo giao diện người dùng (Main Thread).

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình làm việc (Workflow) điển hình của GeoAI diễn ra qua 5 giai đoạn:

1.  **Discovery (Khám phá):** Người dùng sử dụng module `data` để tìm kiếm ảnh vệ tinh trên STAC (Microsoft Planetary Computer) hoặc tải dữ liệu từ NAIP/Overture.
2.  **Preparation (Chuẩn bị):** 
    *   Cắt ảnh vệ tinh lớn thành các Image Chips.
    *   Chuyển đổi dữ liệu vector (labels) sang dạng mặt nạ (masks) hoặc bounding boxes.
    *   Lưu trữ cấu hình trong tệp `project.json`.
3.  **Training/Fine-tuning:** Gọi các hàm `train_*` để huấn luyện mô hình dựa trên các kiến trúc có sẵn (như SAM hoặc Mask R-CNN).
4.  **Inference (Suy luận):** 
    *   Nạp mô hình đã huấn luyện hoặc pre-trained.
    *   Chạy `smooth_inference` trên ảnh mục tiêu.
    *   Tạo ra kết quả dạng Raster (pixel-based).
5.  **Post-processing & Export:**
    *   **Vectorization:** Chuyển đổi mặt nạ pixel thành các đa giác (Polygon).
    *   **Regularization:** Làm phẳng/vuông góc các cạnh của đa giác (đặc biệt hữu ích cho việc trích xuất dấu chân tòa nhà - building footprints).
    *   **Visualization:** Hiển thị kết quả lên bản đồ tương tác qua Leafmap.

### Tổng kết
GeoAI là một framework **High-level API** mạnh mẽ. Nó không cố gắng thay thế các thư viện như PyTorch hay GDAL mà đóng vai trò là một "nhạc trưởng" (Orchestrator) kết nối chúng lại, cung cấp một giao diện dễ dùng cho các nhà khoa học dữ liệu địa lý và các trợ lý AI thông minh.