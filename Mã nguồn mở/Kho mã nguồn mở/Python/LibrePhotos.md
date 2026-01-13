Chào bạn, đây là bản phân tích chi tiết về dự án **LibrePhotos** dựa trên mã nguồn bạn đã cung cấp.

---

### 1. Phân tích Công nghệ Cốt lõi (Tech Stack)

LibrePhotos là một hệ thống quản lý ảnh tự lưu trữ (self-hosted) với kiến trúc hiện đại, kết hợp giữa Web Framework truyền thống và các mô hình Học máy (Machine Learning).

*   **Ngôn ngữ & Framework chính:** Python 3.11+, **Django 5.2.x** (phiên bản rất mới) và **Django REST Framework (DRF)** cho các API.
*   **Cơ sở dữ liệu:** PostgreSQL (sử dụng tính năng `JSONField` và `ArrayField` mạnh mẽ) cùng với **Redis** hoặc DB làm broker cho hàng đợi công việc.
*   **Xử lý tác vụ nền:** **django-q2**. Đây là lựa chọn thay thế cho Celery, giúp quản lý các công việc nặng như scan ảnh, nhận diện khuôn mặt một cách trực quan qua Database.
*   **Xử lý Đồ họa & Media:** 
    *   **PyVips:** Dùng để tạo thumbnail hiệu năng cao (nhanh và ít tốn RAM hơn Pillow).
    *   **FFmpeg:** Xử lý video và tạo thumbnail động (animated thumbnails).
    *   **ExifTool:** Công cụ mạnh mẽ nhất hiện nay để đọc/ghi metadata (EXIF/XMP).
*   **Học máy (Machine Learning) & AI:**
    *   **Face Recognition:** Thư viện `face_recognition` (dựa trên dlib) để tìm khuôn mặt.
    *   **Phân cụm (Clustering):** HDBSCAN và Scikit-learn để nhóm các khuôn mặt của cùng một người.
    *   **Tìm kiếm ngữ nghĩa (Semantic Search):** Sử dụng mô hình **CLIP** (OpenAI) để hiểu nội dung ảnh.
    *   **Indexing:** **FAISS (Facebook AI Similarity Search)** để tìm kiếm ảnh tương đồng trong không gian vector cực nhanh.
    *   **LLM & Captioning:** Tích hợp **Moondream** (LLM đa phương thức nhẹ) và Mistral thông qua `llama-cpp-python`.

---

### 2. Kỹ thuật và Tư duy Kiến trúc

**Kiến trúc Hybrid (Monolith + Microservices):**
Dù mã nguồn nằm trong một repo, LibrePhotos vận hành theo kiểu "Sidecar Microservices". Các dịch vụ nặng về AI (CLIP, Face, Captioning) được tách thành các Flask app riêng biệt chạy trên các cổng khác nhau (8002-8011). Điều này giúp:
*   Tránh việc Python Global Interpreter Lock (GIL) làm nghẽn API chính khi AI đang xử lý.
*   Dễ dàng container hóa và cô lập môi trường cho từng model ML.

**Quản lý tiến trình (Long Running Jobs):**
Hệ thống sử dụng model `LongRunningJob` để theo dõi mọi thứ. Mỗi khi người dùng bắt đầu "Scan", một UUID được tạo ra, cho phép frontend tracking tiến độ (%) theo thời gian thực.

**Tối ưu hóa truy vấn (Performance Tuning):**
Dự án sử dụng một kỹ thuật rất hay là `OptimizeRelatedModelViewSetMetaclass`. Nó tự động phân tích Serializer của DRF để thêm `select_related` và `prefetch_related` vào QuerySet, giúp tránh lỗi "N+1 query" phổ biến trong Django.

---

### 3. Các Kỹ thuật Nổi bật & Sáng tạo

1.  **Phát hiện ảnh trùng lặp (Duplicate Detection):** 
    *   Sử dụng **Perceptual Hashing (pHash)**: Không giống như MD5 (thay đổi 1 bit là hash khác hoàn toàn), pHash tạo ra hash tương tự nhau nếu ảnh trông giống nhau.
    *   Sử dụng cấu trúc dữ liệu **BK-Tree (Burkhard-Keller Tree)**: Cho phép tìm kiếm ảnh tương đồng với độ phức tạp $O(\log N)$ thay vì so sánh từng cặp $O(N^2)$. Điều này cực kỳ quan trọng khi thư viện ảnh lên đến hàng chục nghìn tấm.

2.  **Xử lý Motion Photos:** 
    *   Có logic riêng (`embedded_media.py`) để trích xuất video MP4 ẩn bên trong các tệp JPEG của Google Pixel hoặc Samsung Motion Photos bằng cách đọc trực tiếp binary và tìm kiếm marker (`MotionPhoto_Data`).

3.  **Hệ thống quy định ngày tháng (Date/Time Extractor):** 
    *   LibrePhotos không chỉ tin vào EXIF. Nó có một hệ thống "Rules" ưu tiên: Nếu EXIF sai, nó tìm trong tên tệp (regex), nếu không có thì dùng ngày tạo tệp (mtime), hoặc cho phép người dùng định nghĩa quy tắc riêng.

4.  **Reverse Geocoding đa tầng:** 
    *   Hỗ trợ Mapbox, OpenCage, TomTom và Nominatim. Dữ liệu địa lý được lưu dưới dạng JSON phân cấp giúp tạo ra tính năng "Địa điểm" (Sunburst map) rất đẹp.

---

### 4. Tóm tắt luồng hoạt động (Workflow)

1.  **Quét (Scanning):** `directory_watcher.py` duyệt qua các thư mục -> Tính toán Hash (MD5) để định danh -> Lưu vào DB `File` và `Photo`.
2.  **Trích xuất (Extraction):** Trích xuất EXIF -> Tạo Thumbnail (Big, Small, Tiny) bằng PyVips -> Tính pHash.
3.  **AI xử lý (AI Processing):**
    *   Đưa ảnh qua model **CLIP** để lấy vector đặc trưng (Embeddings) phục vụ tìm kiếm bằng văn bản (vd: gõ "con chó trên bãi biển" sẽ ra ảnh dù ảnh không có tag đó).
    *   Chạy nhận diện khuôn mặt -> Lưu vị trí (bounding box) -> Chạy **HDBSCAN** để tự động gom nhóm những khuôn mặt giống nhau vào một "Person".
4.  **Phân loại (Classification):** Sử dụng Places365 để gán nhãn khung cảnh (phòng khách, bờ biển, rừng rậm).
5.  **Lập chỉ mục (Indexing):** Đẩy các vector vào **FAISS index** để sẵn sàng cho tìm kiếm ngữ nghĩa.

### Đánh giá chung
Đây là một dự án có độ phức tạp kỹ thuật cao, mã nguồn được tổ chức sạch sẽ (tuân thủ Ruff, Type hints). Tư duy sử dụng **BK-Tree** cho duplicate detection và **FAISS** cho tìm kiếm vector cho thấy tác giả rất quan tâm đến khả năng mở rộng (scalability) khi người dùng có thư viện ảnh khổng lồ.

Nếu bạn định đóng góp (contribute), hãy lưu ý chạy các lệnh qua `docker exec` như hướng dẫn trong `CLAUDE.md` vì hệ thống phụ thuộc rất nhiều vào các thư viện C/C++ và các service AI chạy ngầm.