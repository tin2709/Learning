Dựa trên nội dung mã nguồn của dự án **Bracket**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Backend:**
    *   **Ngôn ngữ:** Python 3.12+ (sử dụng tính năng `asyncio` triệt để).
    *   **Framework:** **FastAPI** - tận dụng tốc độ cao, hỗ trợ Type Hinting và tự động tạo tài liệu OpenAPI.
    *   **Database Tooling:** 
        *   **SQLAlchemy** kết hợp với **databases** library để thực hiện các truy vấn không đồng bộ (asynchronous).
        *   **Alembic** để quản lý các phiên bản migration cơ sở dữ liệu.
    *   **Validation:** **Pydantic v2** dùng để định nghĩa schema và kiểm tra dữ liệu đầu vào/đầu ra.
    *   **Authentication:** **JWT (JSON Web Tokens)** và **bcrypt** để băm mật khẩu.
*   **Frontend:**
    *   **Framework:** **React** với **TypeScript**.
    *   **Build Tool:** **Vite** - giúp khởi động và đóng gói ứng dụng cực nhanh.
    *   **UI Library:** **Mantine UI** - cung cấp các component giao diện hiện đại, dễ tùy biến.
    *   **Internationalization:** **i18next** kết hợp với **Crowdin** để hỗ trợ đa ngôn ngữ.
*   **Cơ sở dữ liệu:** **PostgreSQL**.
*   **Infrastructure:** **Docker** và **Docker Compose**, sử dụng **uv** làm trình quản lý package Python (thay thế pip/poetry để tối ưu tốc độ).

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Engineering)

*   **Kiến trúc Phân lớp (Layered Architecture):**
    *   **Routes:** Định nghĩa các endpoint API (`backend/bracket/routes`).
    *   **Logic:** Chứa các nghiệp vụ phức tạp về giải đấu như tính điểm ELO, xếp lịch thi đấu Swiss (`backend/bracket/logic`).
    *   **SQL:** Tách biệt các câu lệnh truy vấn SQL ra khỏi route để dễ bảo trì (`backend/bracket/sql`).
    *   **Models:** Sử dụng Pydantic để phân tách rõ ràng giữa dữ liệu trong DB, dữ liệu trả về và dữ liệu đầu vào.
*   **Xử lý bất đồng bộ (Async Everywhere):** Toàn bộ luồng từ route đến truy vấn DB đều là `async/await`, giúp hệ thống chịu tải tốt hơn khi có nhiều kết nối đồng thời.
*   **Dependency Injection:** FastAPI dependency injection được sử dụng mạnh mẽ để kiểm tra quyền truy cập (Auth), kiểm tra trạng thái giải đấu (ví dụ: không cho sửa giải đấu đã đóng).
*   **Migration-Driven Development:** Sử dụng Alembic để đảm bảo cấu trúc DB đồng bộ trên tất cả các môi trường phát triển và sản xuất.

### 3. Các kỹ thuật chính nổi bật (Highlighted Key Techniques)

*   **Thuật toán Giải đấu phức tạp:**
    *   **Swiss System:** Hỗ trợ tính toán đối thủ dựa trên trình độ tương đương (ELO) và số trận đã đấu.
    *   **Conflict Detection:** Thuật toán phát hiện xung đột trận đấu (vận động viên bị trùng lịch, sân bị trùng lịch) trong `logic/planning/conflicts.py`.
*   **Quản lý Sân thi đấu (Court Scheduling):** Tự động gán sân và tính toán thời gian bắt đầu trận đấu dựa trên thời lượng dự kiến và khoảng nghỉ (`margin_minutes`).
*   **Hệ thống Phân quyền (Access Control):** Phân chia rõ ràng giữa Owner (chủ sở hữu câu lạc bộ) và Collaborator (người cộng tác).
*   **Dynamic Dashboard:** Hỗ trợ tạo các đường dẫn Dashboard công khai (`dashboard_endpoint`) giúp người xem có thể theo dõi kết quả trực tuyến mà không cần đăng nhập.
*   **Cronjobs nội bộ:** Hệ thống tự động xóa các tài khoản Demo hết hạn thông qua `cronjobs/scheduling.py`.

### 4. Tóm tắt luồng hoạt động (Project Flow)

Luồng xử lý một yêu cầu (ví dụ: Cập nhật kết quả trận đấu):

1.  **Frontend:** Người dùng nhập điểm trên giao diện React (Mantine). `services/match.tsx` sẽ gọi API tương ứng.
2.  **API Route:** `routes/matches.py` tiếp nhận request, sử dụng `user_authenticated_for_tournament` để kiểm tra quyền.
3.  **Validation:** Pydantic kiểm tra định dạng dữ liệu trong `match_body`.
4.  **Business Logic:** 
    *   Gọi `logic/ranking/calculation.py` để tính toán lại điểm ELO và bảng xếp hạng ngay khi có kết quả.
    *   Nếu là giải đấu Elimination (loại trực tiếp), gọi `logic/ranking/elimination.py` để tự động đẩy đội thắng vào vòng tiếp theo.
    *   Gọi `logic/planning/conflicts.py` để kiểm tra xem việc thay đổi thời gian có gây xung đột lịch thi đấu không.
5.  **Database:** `sql/matches.py` thực thi lệnh UPDATE vào PostgreSQL thông qua thư viện `databases`.
6.  **Response:** Trả về kết quả JSON thành công, Frontend cập nhật lại trạng thái giao diện.

### 5. Cấu trúc thư mục quan trọng

*   `/backend/bracket/logic/`: "Bộ não" của ứng dụng (thuật toán xếp lịch, tính điểm).
*   `/backend/bracket/models/db/`: Định nghĩa cấu trúc dữ liệu.
*   `/backend/bracket/routes/`: Các cổng giao tiếp API.
*   `/frontend/src/pages/`: Chứa các trang giao diện (Dashboard, Quản lý giải đấu, Cài đặt).
*   `/docs/`: Chứa tài liệu hướng dẫn sử dụng và triển khai (Docusaurus).

Dự án này là một ví dụ điển hình về việc xây dựng ứng dụng Web hiện đại, an toàn và có khả năng mở rộng tốt bằng Python và React.