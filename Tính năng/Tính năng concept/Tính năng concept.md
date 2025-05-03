# 1  So Sánh OAuth và Xác Thực Dựa Trên Phiên (Session-Based Authentication)

## Giới thiệu

Hiểu rõ cách thức hoạt động của các cơ chế xác thực là rất quan trọng khi xây dựng ứng dụng. Bài viết này tập trung so sánh giữa phương pháp xác thực dựa trên phiên làm việc (session-based) truyền thống và phương pháp dựa trên token (cụ thể là OAuth với Access Token và Refresh Token).

## 1. Xác thực Dựa Trên Phiên (Session-Based Authentication)

Đây là phương pháp truyền thống thường được sử dụng trong các ứng dụng web.

**Cách hoạt động:**

1.  **Đăng nhập:** Người dùng cung cấp thông tin đăng nhập (ví dụ: username, password).
2.  **Tạo Phiên:** Nếu thông tin hợp lệ, máy chủ tạo một bản ghi phiên (session) duy nhất và lưu trữ nó ở phía máy chủ (thường trong bộ nhớ cache như Redis hoặc cơ sở dữ liệu).
3.  **Gửi Session ID:** Máy chủ gửi một định danh phiên (Session ID) về cho trình duyệt của người dùng, thường được lưu trữ trong một cookie.
4.  **Yêu cầu Tiếp theo:** Với mỗi yêu cầu tiếp theo đến máy chủ, trình duyệt tự động gửi kèm cookie chứa Session ID.
5.  **Xác thực:** Máy chủ nhận yêu cầu, lấy Session ID từ cookie, tìm kiếm và xác thực phiên tương ứng trong bộ nhớ lưu trữ của nó. Nếu phiên hợp lệ, yêu cầu được xử lý.

**Đặc điểm:**

*   **Stateful:** Máy chủ cần duy trì trạng thái phiên của mỗi người dùng đang hoạt động.
*   **Phụ thuộc Cookie:** Thường dựa vào cơ chế cookie của trình duyệt.
*   **Triển khai:** Tương đối đơn giản trong các framework web truyền thống.

## 2. Xác thực Dựa Trên Token (OAuth / JWT)

Phương pháp này hiện đại hơn, linh hoạt và thường được sử dụng trong các API, ứng dụng trang đơn (SPA), và ứng dụng di động. OAuth 2.0 là một framework phổ biến sử dụng cơ chế này.

**Các thành phần chính:**

*   **Access Token:** Một chuỗi mã thông báo (thường là JWT - JSON Web Token) có thời hạn ngắn, cấp quyền truy cập tài nguyên cụ thể cho người dùng trong một khoảng thời gian giới hạn.
*   **Refresh Token:** Một chuỗi mã thông báo có thời hạn dài hơn, được sử dụng để yêu cầu một Access Token mới khi Access Token cũ hết hạn mà không cần người dùng đăng nhập lại.

**Cách hoạt động (Luồng OAuth 2.0 phổ biến - Authorization Code Grant):**

1.  **Ủy quyền:** Ứng dụng chuyển hướng người dùng đến trang đăng nhập của nhà cung cấp dịch vụ (ví dụ: Google, Facebook) cùng với `client_id` (định danh ứng dụng) và `redirect_uri` (nơi trả kết quả về).
2.  **Đăng nhập & Chấp thuận:** Người dùng đăng nhập và cấp quyền cho ứng dụng truy cập thông tin của họ.
3.  **Nhận Authorization Code:** Nhà cung cấp dịch vụ chuyển hướng người dùng trở lại `redirect_uri` của ứng dụng kèm theo một `authorization_code`.
4.  **Đổi Code lấy Token:** Backend của ứng dụng gửi `authorization_code`, `client_id`, và `client_secret` đến endpoint token của nhà cung cấp dịch vụ.
    ```http
    POST /token HTTP/1.1
    Host: oauth-provider.com
    Content-Type: application/x-www-form-urlencoded

    grant_type=authorization_code
    &code=AUTHORIZATION_CODE
    &redirect_uri=YOUR_REDIRECT_URI
    &client_id=YOUR_CLIENT_ID
    &client_secret=YOUR_CLIENT_SECRET
    ```
5.  **Nhận Tokens:** Nhà cung cấp dịch vụ xác thực thông tin và trả về `access_token` và `refresh_token`.
    ```json
    {
      "access_token": "ACCESS_TOKEN_STRING",
      "token_type": "Bearer",
      "expires_in": 3600, // Thời gian sống của access token (giây)
      "refresh_token": "REFRESH_TOKEN_STRING",
      "scope": "requested_scopes"
    }
    ```
6.  **Sử dụng Access Token:** Ứng dụng sử dụng `access_token` để gọi các API được bảo vệ của nhà cung cấp dịch vụ hoặc API của chính ứng dụng. Token thường được gửi trong header `Authorization`.
    ```http
    GET /api/userinfo HTTP/1.1
    Host: resource-server.com
    Authorization: Bearer ACCESS_TOKEN_STRING
    ```
7.  **Làm mới Access Token:** Khi `access_token` hết hạn, ứng dụng sử dụng `refresh_token` để yêu cầu một `access_token` mới từ endpoint token mà không cần người dùng tương tác lại.
    ```http
    POST /token HTTP/1.1
    Host: oauth-provider.com
    Content-Type: application/x-www-form-urlencoded

    grant_type=refresh_token
    &refresh_token=REFRESH_TOKEN_STRING
    &client_id=YOUR_CLIENT_ID
    &client_secret=YOUR_CLIENT_SECRET
    ```
8.  **Nhận Access Token Mới:** Nhà cung cấp dịch vụ trả về một `access_token` mới (và đôi khi cả `refresh_token` mới).

**Tại sao cần Access Token và Refresh Token?**

*   **Access Token (Ngắn hạn):**
    *   Giảm thiểu rủi ro bảo mật. Nếu token bị lộ, kẻ tấn công chỉ có quyền truy cập trong thời gian ngắn.
    *   Cấp quyền truy cập tạm thời, có thể chứa thông tin về phạm vi quyền (scopes).
*   **Refresh Token (Dài hạn):**
    *   Cải thiện trải nghiệm người dùng (UX) bằng cách cho phép duy trì đăng nhập trong thời gian dài mà không cần nhập lại mật khẩu thường xuyên.
    *   Được lưu trữ an toàn hơn (thường là phía backend hoặc trong `HttpOnly` cookie) và chỉ dùng cho mục đích lấy Access Token mới.

## 3. So Sánh Chính

| Tính năng             | Xác thực Dựa trên Phiên (Session-Based)          | OAuth / Token-Based                                     |
| :-------------------- | :----------------------------------------------- | :------------------------------------------------------ |
| **Trạng thái**        | **Stateful** (Máy chủ lưu trạng thái phiên)       | **Thường là Stateless** (Máy chủ không cần lưu phiên)   |
| **Lưu trữ**           | Session ID trong Cookie, dữ liệu phiên trên Server | Tokens lưu trữ phía Client (Local Storage, Session Storage, Cookie) |
| **Khả năng mở rộng** | Khó hơn khi scale ngang (cần chia sẻ session)     | Dễ dàng scale ngang hơn                              |
| **CSRF**              | Cần cơ chế phòng chống (ví dụ: CSRF token)        | Ít bị ảnh hưởng nếu không dùng cookie để lưu token      |
| **Nền tảng**          | Chủ yếu cho Web truyền thống                     | Tốt cho Web (SPA), Mobile, APIs, Server-to-Server       |
| **Third-party Auth**  | Khó tích hợp trực tiếp                           | Là tiêu chuẩn (OAuth) cho việc ủy quyền bên thứ ba     |
| **Bảo mật Token**     | Không áp dụng                                   | Access Token ngắn hạn giảm rủi ro, Refresh Token cần bảo mật cẩn thận |

## 4. Lưu ý: Refresh Token KHÁC Access Key / Secret Key

*   **Refresh Token:** Dùng để gia hạn **phiên đăng nhập của người dùng**.
*   **Access Key / Secret Key:** Là cặp thông tin xác thực **dài hạn** (giống username/password) dùng để xác thực **máy móc, dịch vụ, hoặc ứng dụng** (ví dụ: khi ứng dụng của bạn gọi API AWS S3, DynamoDB). Thường dùng cho giao tiếp server-to-server.

## 5. Kết luận

Việc lựa chọn giữa xác thực dựa trên phiên và OAuth/token phụ thuộc vào yêu cầu cụ thể của ứng dụng:

*   **Session-Based:** Phù hợp cho các ứng dụng web đơn giản, truyền thống, nơi không cần tích hợp phức tạp hoặc chia sẻ xác thực qua nhiều nền tảng.
*   **OAuth/Token-Based:** Linh hoạt và mạnh mẽ hơn, đặc biệt phù hợp cho các API, ứng dụng di động, SPA, microservices, và khi cần tích hợp đăng nhập/ủy quyền với các dịch vụ bên thứ ba.

Hiểu rõ bản chất và sự khác biệt của từng phương pháp giúp đưa ra quyết định kiến trúc đúng đắn, đảm bảo cả tính bảo mật và trải nghiệm người dùng tốt.

![alt text](image.png)