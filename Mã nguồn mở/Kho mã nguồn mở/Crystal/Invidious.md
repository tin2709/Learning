Dựa trên các tệp mã nguồn và cấu trúc thư mục của dự án **Invidious**, dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và cơ chế vận hành của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Invidious nổi bật nhờ việc sử dụng các công nghệ tối ưu cho hiệu suất cao và quyền riêng tư:

*   **Ngôn ngữ lập trình Crystal:** Đây là yếu tố quan trọng nhất. Crystal có cú pháp giống Ruby (dễ viết, dễ đọc) nhưng tốc độ thực thi tương đương C/C++ nhờ trình biên dịch LLVM. Nó cho phép xử lý hàng nghìn kết nối đồng thời với mức tiêu thụ RAM cực thấp.
*   **Kemal Framework:** Một web framework siêu nhẹ (micro-framework) dành cho Crystal, đóng vai trò điều hướng (routing) và xử lý các request HTTP.
*   **Cơ sở dữ liệu PostgreSQL:** Sử dụng để lưu trữ thông tin người dùng, cài đặt cá nhân, danh sách đăng ký (subscriptions) và bộ nhớ đệm (cache) cho siêu dữ liệu video.
*   **Video.js:** Thư viện trình phát video chính ở frontend, được tùy chỉnh sâu để hỗ trợ các luồng DASH và các tính năng như thay đổi tốc độ, phụ đề mà không cần đến script của Google.
*   **ECR (Embedded Crystal):** Engine tạo template phía server, giúp render HTML nhanh chóng trước khi gửi đến trình duyệt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Invidious được thiết kế theo triết lý **"Middleman & Decoupling"** (Người trung gian và Phân tách):

*   **Kiến trúc không API chính thức:** Invidious không sử dụng API chính thức của YouTube (Google Cloud Console). Thay vào đó, nó đóng vai trò là một "browser giả lập", trích xuất dữ liệu trực tiếp từ các endpoint nội bộ của YouTube (InnerTube API) hoặc phân tích mã nguồn trang web. Điều này giúp tránh bị giới hạn quota và yêu cầu khóa API.
*   **Invidious Companion (Kiến trúc phân tán):** Gần đây, dự án tách phần xử lý chữ ký số (signature) và vượt qua cơ chế kiểm tra bot của YouTube ra một module riêng gọi là `invidious-companion`. Điều này giúp ứng dụng chính nhẹ hơn và dễ dàng thay đổi logic khi YouTube thay đổi thuật toán bảo vệ.
*   **Cơ chế Proxy hóa (Privacy Proxy):** Mọi request từ người dùng đến YouTube (ảnh thumnail, luồng video) đều đi qua server Invidious. Server này "rửa" sạch các định danh cá nhân, giúp YouTube chỉ thấy IP của server thay vì IP của người dùng cuối.
*   **Thiết kế Stateless & Cached:** Hệ thống cố gắng lưu trữ tạm thời (cache) các phản hồi từ YouTube vào DB (bảng `videos` được đánh dấu là `UNLOGGED` trong Postgres để tăng tốc độ ghi) nhằm giảm tải cho mạng và tăng tốc độ phản hồi.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Xử lý luồng bất đồng bộ (Fibers):** Tận dụng mô hình Concurrency của Crystal (giống Go-routines) để thực hiện hàng loạt các request trích xuất dữ liệu từ YouTube mà không làm nghẽn luồng xử lý chính.
*   **Regular Expression & Protobuf Parsing:** Vì YouTube trả về dữ liệu dưới dạng JSON phức tạp hoặc Protobuf nén, Invidious sử dụng các bộ lọc (`extractors.cr`) cực mạnh để bóc tách thông tin từ các chuỗi dữ liệu hỗn loạn.
*   **Custom Static File Handler:** Thay vì dùng handler mặc định, dự án viết lại logic xử lý file tĩnh (`kemal_static_file_handler.cr`) để tối ưu hóa bộ nhớ đệm và bảo mật header.
*   **Internationalization (i18n):** Sử dụng hệ thống tệp JSON trong `locales/` kết hợp với logic Crystal để hỗ trợ đa ngôn ngữ mà không làm giảm hiệu suất render.
*   **Database Migrations tự thân:** Dự án tự quản lý việc nâng cấp schema cơ sở dữ liệu thông qua các script SQL và code Crystal trong `src/invidious/database/migrations/`.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng xem Video (Watch Flow):
1.  **Request:** Người dùng truy cập `/watch?v=VIDEO_ID`.
2.  **Metadata Fetch:** Invidious kiểm tra trong DB local. Nếu không có hoặc hết hạn, nó gửi request đến YouTube (giả lập thiết bị Android hoặc Web).
3.  **Extraction:** Bộ trích xuất (`parser.cr`) lấy ra link stream trực tiếp, phụ đề và thông tin kênh.
4.  **Proxying:** Nếu người dùng bật chế độ "Proxy video", Invidious sẽ tạo ra các link `/videoplayback` trỏ về chính nó. Khi trình duyệt yêu cầu dữ liệu, Invidious sẽ tải từng đoạn video từ YouTube rồi chuyển tiếp ngay lập tức cho người dùng (Stream pipe).
5.  **Rendering:** Trả về HTML đã render qua ECR kèm trình phát Video.js.

#### B. Luồng tìm kiếm (Search Flow):
1.  Người dùng nhập từ khóa.
2.  Invidious gửi request tìm kiếm đến YouTube.
3.  Hệ thống lọc bỏ toàn bộ quảng cáo và các thành phần theo dõi của Google.
4.  Dữ liệu được cấu trúc lại thành một danh sách JSON/HTML sạch sẽ và hiển thị cho người dùng.

#### C. Luồng làm mới dữ liệu (Background Jobs):
*   Một hệ thống các công việc chạy nền (`jobs/`) liên tục thực hiện các nhiệm vụ:
    *   `refresh_channels_job`: Cập nhật video mới cho các kênh mà người dùng đã đăng ký.
    *   `clear_expired_items_job`: Xóa bỏ các session, nonce và cache cũ trong database để giải phóng dung lượng.

### Tổng kết
Invidious là một kỳ công về kỹ thuật **Reverse Engineering** (Kỹ thuật đảo ngược). Nó biến một nền tảng đóng như YouTube thành một dịch vụ có thể tự vận hành (self-hosted), đề cao quyền riêng tư bằng cách sử dụng hiệu năng vượt trội của ngôn ngữ Crystal và chiến thuật "đứng giữa" để bảo vệ người dùng.