Chào bạn, đây là bản phân tích chi tiết dự án **NeoDB** dựa trên mã nguồn và cấu trúc tệp tin mà bạn đã cung cấp.

---

# Phân tích Hệ thống NeoDB

**NeoDB** (tiền thân là Boofilsic) là một nền tảng mã nguồn mở giúp quản lý, đánh giá và chia sẻ các sản phẩm văn hóa (sách, phim, nhạc, game, podcast). Điểm đặc biệt nhất là nó được thiết kế để hoạt động trong mạng lưới **Fediverse** thông qua giao thức ActivityPub và ATProto.

## 1. Công nghệ cốt lõi (Core Technology)

Dựa trên `pyproject.toml` và `Dockerfile`, NeoDB sử dụng một stack công nghệ hiện đại và mạnh mẽ:

*   **Ngôn ngữ & Framework:** **Python 3.13** và **Django 5.2**. Việc sử dụng Django làm khung sườn giúp tận dụng hệ thống ORM, Admin và quản lý User cực kỳ ổn định.
*   **Cơ sở dữ liệu:** **PostgreSQL**. Dự án sử dụng hai DB riêng biệt (một cho NeoDB core và một cho Takahē - engine liên hợp).
*   **Caching & Task Queue:** **Redis** kết hợp với **django-rq**. Đây là bộ phận quan trọng để xử lý các tác vụ nền như cào dữ liệu (scraping), đồng bộ hóa mạng xã hội và xử lý federation.
*   **Search Engine:** **Typesense**. Một công cụ tìm kiếm mã nguồn mở tốc độ cao, dùng để tìm kiếm nhanh trong catalog hàng triệu sản phẩm.
*   **API:** **Django Ninja**. Framework này giúp tạo các RESTful API dựa trên Type hints của Python, cho hiệu suất cao và tự động tạo tài liệu OpenAPI.
*   **Liên hợp (Federation):** Dựa trên **Takahē** (một triển khai ActivityPub bằng Django) để giao tiếp với Mastodon, Pleroma... và thư viện **atproto** để giao tiếp với Bluesky.
*   **Xử lý dữ liệu & Scraping:** Sử dụng `httpx`, `lxml`, `beautifulsoup4` để lấy dữ liệu từ các trang như IMDb, TMDB, Douban.

## 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của NeoDB là sự kết hợp giữa **Monolith (Django)** và **Asynchronous Workers (RQ Workers)**:

*   **Kiến trúc Đa hình (Polymorphism):** Sử dụng `django-polymorphic`. Lớp `Item` là lớp cha cho tất cả các loại sản phẩm (Book, Movie, Game...). Điều này giúp quản lý catalog chung nhưng vẫn giữ được các thuộc tính riêng biệt cho từng loại sản phẩm.
*   **Tách biệt Catalog và Journal:**
    *   `catalog/`: Quản lý kho dữ liệu chung (thông tin khách quan về phim, sách).
    *   `journal/`: Quản lý dữ liệu cá nhân (đánh giá, ghi chú, trạng thái "đang xem/đã xem").
*   **Tích hợp Takahē làm Sub-engine:** Thay vì tự viết lại từ đầu giao thức ActivityPub phức tạp, NeoDB tích hợp Takahē để biến mỗi instance NeoDB thành một node trong Fediverse.
*   **Cấu trúc Proxy/Adapter cho Scraping:** Mỗi trang web bên thứ ba (Amazon, Steam, IMDb) có một "Site Class" riêng trong `catalog/sites/`, giúp dễ dàng mở rộng thêm các nguồn dữ liệu mới mà không ảnh hưởng đến lõi hệ thống.

## 3. Các kỹ thuật then chốt (Key Techniques)

*   **Mapping Định danh (ExternalResource & LookupID):** Một sản phẩm có thể có nhiều ID (ISBN cho sách, IMDb ID cho phim, ASIN cho Amazon). NeoDB sử dụng bảng `ExternalResource` để map tất cả các ID này về cùng một `Item` duy nhất trong hệ thống, tránh trùng lặp.
*   **Internationalization (i18n) sâu:** Sử dụng `django-jsonform` để lưu trữ tiêu đề và mô tả sản phẩm bằng nhiều ngôn ngữ khác nhau trong cùng một trường JSON (trường `localized_title`).
*   **Xử lý bất đồng bộ (Background Tasks):** Các công việc nặng như `PodcastUpdater` (cập nhật RSS podcast) hay `DiscoverGenerator` (tạo danh sách xu hướng) được lập lịch chạy định kỳ thông qua cron/rqworker.
*   **Middleware nhận diện danh tính:** `common.middleware.IdentityMiddleware` cho phép hệ thống nhận diện người dùng đến từ các instance khác nhau trong mạng lưới liên hợp.

## 4. Tóm tắt luồng hoạt động (Flow Summary)

Dưới đây là luồng hoạt động chính của một số tính năng quan trọng:

### A. Luồng tạo sản phẩm mới (Fetch/Crawl Flow):
1.  Người dùng dán một link (vídụ: link IMDb).
2.  `SiteManager` nhận diện URL và gọi Scraper tương ứng.
3.  Hệ thống kiểm tra `ExternalResource` xem đã tồn tại ID này chưa.
4.  Nếu chưa, một task `RQ` được đẩy vào hàng đợi để cào dữ liệu ngầm.
5.  Sau khi cào xong, dữ liệu được chuẩn hóa (normalize) và lưu vào `Item` kèm theo cover image.
6.  Sản phẩm được đánh chỉ mục (index) vào `Typesense` để người khác có thể tìm thấy.

### B. Luồng tương tác xã hội (Social Flow):
1.  Người dùng viết Review cho một cuốn sách.
2.  Dữ liệu lưu vào bảng `Review` trong app `journal`.
3.  Hệ thống gọi Takahē để đóng gói Review này thành một `Activity` (ví dụ: Create Note hoặc Announce).
4.  Task `ap` (ActivityPub) sẽ gửi Activity này tới Inbox của các máy chủ Mastodon/Pleroma mà những người theo dõi (followers) đang sử dụng.

### C. Luồng tìm kiếm (Search Flow):
1.  Người dùng nhập từ khóa.
2.  Django Ninja API nhận yêu cầu và gọi `CatalogIndex`.
3.  Yêu cầu được chuyển đến `Typesense server` với các bộ lọc về Category (Book, Movie...).
4.  Kết quả trả về được map lại với các model Django để hiển thị giao diện kèm theo trạng thái cá nhân (ví dụ: "Bạn đã xem phim này").

---
**Kết luận:** NeoDB là một dự án có kỹ thuật cao, giải quyết tốt bài toán **quản lý dữ liệu tập trung** trong một **môi trường phi tập trung (decentralized)**. Việc tận dụng tối đa hệ sinh thái Django và kiến trúc worker giúp nó có khả năng mở rộng tốt.