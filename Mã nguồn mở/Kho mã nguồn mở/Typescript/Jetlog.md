Dưới đây là phân tích chi tiết về dự án **Jetlog** dựa trên mã nguồn bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Stack)

Dự án sử dụng mô hình **Single Page Application (SPA)** với Backend tách biệt phục vụ qua REST API.

*   **Backend (Python):**
    *   **FastAPI:** Framework hiện đại, tốc độ cao để xây dựng API.
    *   **SQLite:** Cơ sở dữ liệu quan hệ nhẹ, không cần server riêng, phù hợp cho việc tự lưu trữ (self-hosting).
    *   **Pydantic:** Kiểm tra và xác thực dữ liệu (Data validation) cực kỳ chặt chẽ.
    *   **Argon2 & PyJWT:** Sử dụng thuật toán băm mật khẩu mạnh (Argon2) và mã xác thực Token (JWT).
*   **Frontend (TypeScript/React):**
    *   **React:** Thư viện UI để xây dựng giao diện người dùng.
    *   **Tailwind CSS:** Framework CSS utility-first giúp tùy chỉnh giao diện nhanh chóng.
    *   **React Simple Maps:** Thư viện dựa trên D3 để vẽ bản đồ thế giới và đường bay.
    *   **Parcel:** Công cụ đóng gói (bundler) mã nguồn nhanh, cấu hình tối giản.
*   **DevOps & Deployment:**
    *   **Docker & Docker Compose:** Đóng gói toàn bộ ứng dụng để chạy ở bất cứ đâu.
    *   **GitHub Actions:** Tự động hóa việc build và đẩy ảnh (image) lên Docker Hub.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architecture)

*   **Kiến trúc Phân tầng (Layered Architecture):**
    *   **Routers:** Phân chia API theo chức năng (flights, airports, statistics, auth...).
    *   **Models:** Định nghĩa dữ liệu đồng nhất giữa Database, API và Frontend thông qua Pydantic (Backend) và Class (Frontend).
    *   **Internal Utils:** Tách biệt logic tính toán (như công thức Haversine tính khoảng cách) khỏi các route.
*   **Tư duy Self-hosting & Portable:**
    *   Jetlog được thiết kế để "chạy là xong". Nó tự động khởi tạo database SQLite, tự động "vá" (patch) schema nếu có phiên bản mới (`database.py`), và tích hợp sẵn dữ liệu sân bay/hàng không toàn cầu.
*   **Bảo mật:**
    *   Phân quyền rõ ràng: `admin` có quyền quản lý người dùng khác, trong khi người dùng thường chỉ quản lý được chuyến bay của chính mình.
    *   Sử dụng `OAuth2PasswordBearer` để quản lý luồng đăng nhập.

---

### 3. Các kỹ thuật chính nổi bật (Technical Highlights)

1.  **Xử lý dữ liệu địa lý nâng cao:**
    *   **Công thức Haversine:** Trong `server/routers/flights.py`, hệ thống tự tính toán khoảng cách đường chim bay giữa hai sân bay dựa trên vĩ độ và kinh độ.
    *   **Xử lý múi giờ (Timezones):** Hệ thống sử dụng thư viện `pytz` để chuyển đổi thời gian địa phương của sân bay về chuẩn UTC để tính toán thời gian bay chính xác.
2.  **Hệ thống "Vá" Cơ sở dữ liệu (Database Migration/Patching):**
    *   Thay vì dùng công cụ phức tạp như Alembic, tác giả tự viết logic trong `database.py` để kiểm tra các cột bị thiếu và tự động `ALTER TABLE` hoặc sao chép dữ liệu sang bảng tạm để cập nhật schema mà không mất dữ liệu người dùng.
3.  **Tối ưu hóa Docker:**
    *   **Multi-stage Build:** Tách giai đoạn build Frontend (Node.js) và chạy Backend (Python) để giảm kích thước ảnh Docker cuối cùng.
    *   **Permission Management:** Sử dụng `gosu` và `tini` trong `entrypoint.sh` để đảm bảo ứng dụng chạy với quyền người dùng không phải root (`PUID/PGID`), giúp bảo mật tốt hơn trên môi trường NAS hoặc Linux.
4.  **Tích hợp dữ liệu ngoài:**
    *   Hệ thống có khả năng import từ nhiều nguồn: MyFlightRadar24, Flighty và Custom CSV. Logic xử lý CSV (`importing.py`) rất mạnh mẽ, bao gồm việc chuyển đổi định dạng và kiểm tra trùng lặp.
    *   Tích hợp API bên ngoài (`adsbdb`) để tự động điền thông tin chuyến bay dựa trên số hiệu (Flight Number).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Dựa trên các file mã nguồn, quy trình hoạt động của dự án như sau:

1.  **Khởi tạo (Startup):**
    *   Docker chạy `entrypoint.sh`, thiết lập quyền thư mục `/data`.
    *   Backend FastAPI khởi chạy, `database.py` kiểm tra file `jetlog.db`. Nếu chưa có, nó tạo bảng và nạp dữ liệu sân bay từ các file `.db` có sẵn trong thư mục `data`.
2.  **Người dùng tương tác (Frontend):**
    *   Người dùng đăng nhập -> Nhận JWT Token -> Lưu vào `sessionStorage`.
    *   Khi người dùng thêm chuyến bay mới (`New.tsx`), Frontend có thể gọi API bên ngoài để lấy dữ liệu máy bay hoặc tìm kiếm sân bay từ database nội bộ thông qua `SearchInput.tsx`.
3.  **Xử lý dữ liệu (Processing):**
    *   Khi một chuyến bay được lưu, Backend tính toán: Khoảng cách, Thời gian bay (dựa trên múi giờ sân bay), và ghi vào SQLite.
    *   Người dùng có thể yêu cầu "Compute connections" để hệ thống tự động tìm các chuyến bay nối chuyến dựa trên thời gian và địa điểm.
4.  **Hiển thị & Thống kê (Visualization):**
    *   `WorldMap.tsx` lấy dữ liệu tọa độ từ API `/geography/decorations`, vẽ các đường bay (Lines) và điểm dừng (Markers).
    *   Trang Thống kê (`Statistics.tsx`) thực hiện các truy vấn SQL phức tạp (Aggregate functions) để tính tổng quãng đường, số quốc gia đã đi, và các sân bay bận rộn nhất.
5.  **Xuất dữ liệu (Export):**
    *   Người dùng có thể xuất toàn bộ lịch sử bay sang định dạng CSV hoặc file lịch iCal để đồng bộ với Google Calendar/Apple Calendar.

### Đánh giá chung:
Đây là một dự án **Clean Code**, cấu trúc thư mục rõ ràng, áp dụng các kỹ thuật lập trình Web hiện đại và có tư duy sản phẩm hoàn thiện (Product-oriented) rất tốt cho môi trường tự lưu trữ (self-hosted).