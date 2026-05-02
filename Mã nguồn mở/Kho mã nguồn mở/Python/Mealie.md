Dựa trên mã nguồn và cấu trúc thư mục của dự án **Mealie**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và quy trình vận hành của hệ thống quản lý công thức nấu ăn tự lưu trữ (self-hosted) này.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng theo mô hình hiện đại, tách biệt hoàn toàn giữa Backend và Frontend:

*   **Backend (Python 62.6%):**
    *   **FastAPI:** Framework web hiệu năng cao được sử dụng để xây dựng RESTful API. Tận dụng tối đa `async/await` và `Pydantic` để kiểm tra kiểu dữ liệu (validation).
    *   **SQLAlchemy & Alembic:** ORM để tương tác với cơ sở dữ liệu và công cụ quản lý migration (SQLite cho người dùng cá nhân, PostgreSQL cho quy mô lớn).
    *   **Pydantic:** Đóng vai trò trung tâm trong việc định nghĩa các Schema cho request/response.
    *   **Lớp xử lý dữ liệu:** Sử dụng `beautifulsoup4`, `lxml` và `recipe-scrapers` để bóc tách dữ liệu công thức từ các trang web khác nhau.
    *   **AI Integration:** Tích hợp OpenAI API để nhận diện hình ảnh công thức, chuyển đổi video thành văn bản và phân tích nguyên liệu.

*   **Frontend (Vue & TypeScript ~37%):**
    *   **Nuxt.js (Vue 3):** Framework để xây dựng ứng dụng Web đa trang (SPA/Static), tối ưu hóa SEO và trải nghiệm người dùng.
    *   **Vuetify:** Thư viện UI component theo phong cách Material Design giúp giao diện đồng nhất và dễ sử dụng trên di động.
    *   **TypeScript:** Đảm bảo an toàn kiểu dữ liệu từ Backend xuống Frontend (thông qua các script generate types tự động).

*   **Hạ tầng & DevOps:**
    *   **Docker & Docker Compose:** Phương thức triển khai chính, giúp đóng gói toàn bộ môi trường chạy.
    *   **Taskfile:** Sử dụng `Taskfile.yml` thay cho Makefile truyền thống để quản lý các lệnh build, test, và generate code.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Mealie tuân thủ nguyên tắc **Layered Architecture (Kiến trúc phân lớp)** và **Domain-Driven**:

*   **Tách biệt Domain:** Dữ liệu được tổ chức theo các nhóm logic: `Recipe` (Công thức), `Household` (Hộ gia đình), `Group` (Nhóm), `ShoppingList` (Danh sách mua sắm).
*   **Phân lớp Backend:**
    *   `routes`: Nơi tiếp nhận các request HTTP (Controllers).
    *   `services`: Chứa logic nghiệp vụ phức tạp (ví dụ: logic tính toán khẩu phần, bóc tách web).
    *   `repos`: Lớp trừu tượng truy cập dữ liệu (Repositories), giúp tách biệt logic SQL khỏi logic nghiệp vụ.
    *   `schema`: Định nghĩa cấu trúc dữ liệu truyền tải giữa Client và Server.
*   **Multi-tenancy (Đa người dùng):** Dự án hỗ trợ phân cấp từ `Group` -> `Household` -> `User`. Điều này cho phép nhiều gia đình có thể dùng chung một instance Mealie nhưng vẫn giữ riêng tư danh sách đi chợ và kế hoạch bữa ăn.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Automated Code Generation (Sinh mã tự động):** Trong thư mục `dev/code-generation/`, dự án có các script Python dùng để:
    *   Chuyển đổi Pydantic Models (Python) thành TypeScript Interfaces cho Frontend.
    *   Tự động cập nhật các file ngôn ngữ (Locales).
    *   Đảm bảo Frontend luôn khớp với các thay đổi của API mà không cần viết tay lại các định nghĩa Type.
*   **Natural Language Processing (NLP):** Mealie sử dụng thư viện `ingredient-parser-nlp` để tách các chuỗi văn bản tự do (như "2 cups of chopped onions") thành các trường dữ liệu có cấu trúc: Số lượng (2), Đơn vị (cups), Thực phẩm (onions).
*   **Dependency Injection (Tiêm phụ thuộc):** FastAPI được sử dụng để inject cơ sở dữ liệu (`session`), người dùng hiện tại (`current_user`) vào các hàm xử lý API, giúp code dễ kiểm thử (unit test).
*   **Strategy Pattern trong Scraper:** Sử dụng nhiều chiến lược (strategies) khác nhau để lấy dữ liệu từ các trang web (JSON-LD, Microdata, hoặc bóc tách HTML thuần) tùy thuộc vào cấu trúc của trang nguồn.

---

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Quy trình Nhập công thức (Recipe Import):
1.  **Client:** Gửi một URL công thức tới Backend.
2.  **Server (Scraper Service):** Tải nội dung trang web, sử dụng các chiến lược bóc tách dữ liệu để lấy tên món, nguyên liệu, hướng dẫn.
3.  **Server (NLP Parser):** Phân tích danh sách nguyên liệu thô để map vào database `Foods` và `Units` hiện có.
4.  **Database:** Lưu công thức mới và trả về dữ liệu JSON cho Frontend hiển thị.

#### B. Quy trình Lập kế hoạch & Mua sắm (Meal Plan to Shop):
1.  **User:** Chọn các món ăn cho tuần tới trong `Meal Planner`.
2.  **System:** Tổng hợp tất cả nguyên liệu từ các món đã chọn.
3.  **Logic:** Hệ thống tự động gộp các nguyên liệu trùng nhau (ví dụ: 2 món đều cần hành tây sẽ được cộng dồn số lượng).
4.  **Shopping List:** Xuất ra danh sách đi chợ được phân loại theo các quầy hàng (Labels) như "Rau củ", "Đồ khô".

#### C. Quy trình Sao lưu (Backup/Restore):
*   Hệ thống định kỳ hoặc theo lệnh của admin sẽ đóng gói dữ liệu database (JSON) và các tệp hình ảnh vào một file `.zip`.
*   Quy trình khôi phục thực hiện migration dữ liệu từ các phiên bản cũ lên phiên bản mới nhất, đảm bảo tính toàn vẹn của dữ liệu qua các đợt cập nhật phần mềm.

### Tổng kết
Mealie là một dự án quản lý dữ liệu gia đình cực kỳ bài bản. Nó không chỉ là một trang web hiển thị, mà là một **hệ thống xử lý ngôn ngữ tự nhiên và bóc tách dữ liệu** mạnh mẽ, được tối ưu hóa cho việc tự vận hành (self-hosting) với kiến trúc linh hoạt giữa SQLite và PostgreSQL.