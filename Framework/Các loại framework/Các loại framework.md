

## 1 Vaadin Flow vs Hilla: Chọn framework nào?##

Bài viết thảo luận về việc lựa chọn giữa hai framework web của Vaadin: Vaadin Flow và Hilla. Cả hai đều cho phép xây dựng giao diện người dùng (UI) hiện đại với thư viện component phong phú, định tuyến và backend Java. Tác giả nhấn mạnh rằng dù có nhiều điểm chung, trải nghiệm phát triển (developer experience - DX) giữa chúng rất khác nhau và không có cái nào "tốt hơn" một cách tuyệt đối. Việc lựa chọn phụ thuộc vào ngữ cảnh cụ thể của bạn.

**1. Điểm khác biệt cốt lõi: Hướng giải quyết Full Stack**

Cả Vaadin Flow và Hilla đều là framework full-stack, nhưng chúng giải quyết vấn đề full-stack theo hai hướng ngược nhau:

*   **Vaadin Flow (Server-driven - Hướng máy chủ):**
    *   Bạn viết toàn bộ logic UI bằng **Java**.
    *   Framework tự động xử lý giao tiếp client-server (sử dụng XHR hoặc WebSockets), nghĩa là bạn **không cần** phải xây dựng các REST endpoint hay quản lý một dự án frontend riêng biệt.
    *   Ứng dụng chạy trên máy chủ, cho phép nhà phát triển truy cập trực tiếp vào dữ liệu và dịch vụ Java.
    *   HTML, JavaScript và CSS có thể được sử dụng để tùy chỉnh, nhưng không bắt buộc phải có để xây dựng một ứng dụng.
    *   **Ưu điểm chính:** Đơn giản hóa phát triển web, cho phép nhà phát triển tập trung vào logic nghiệp vụ và hoàn toàn làm việc trong môi trường Java.
    *   **Nền tảng công nghệ:** Java-based component API, tự động hóa giao tiếp.

*   **Vaadin Hilla (Client-driven - Hướng client):**
    *   Bạn viết UI bằng **TypeScript**, sử dụng Lit và các Vaadin component.
    *   Backend vẫn là **Java**, nhưng các API của bạn được hiển thị dưới dạng các endpoint an toàn kiểu dữ liệu (type-safe endpoints) mà client sẽ trực tiếp tiêu thụ.
    *   Hilla kết hợp mô hình component dựa trên TypeScript với các template khai báo, phản ứng (reactive, declarative templates) và render DOM hiệu quả.
    *   **Ưu điểm chính:** Hỗ trợ giao tiếp không đồng bộ, an toàn kiểu dữ liệu với backend Java. Tự động tạo định nghĩa kiểu (type definitions) được chia sẻ từ các lớp server, giúp bắt lỗi phá vỡ API ngay tại thời điểm biên dịch thay vì runtime. Lý tưởng cho các nhóm ưu tiên hoặc đã sử dụng các công cụ frontend hiện đại.
    *   **Nền tảng công nghệ:** UI xây dựng bằng Web Components với LitElement và TypeScript. Server export các hàm typed, asynchronous cho client.

**2. Bảng so sánh nhanh (Quick comparison):**

Bài viết cung cấp một bảng tóm tắt giúp lựa chọn:

| Tình huống của bạn                   | Chọn Flow | Chọn Hilla |
| :----------------------------------- | :-------- | :--------- |
| Xây dựng ứng dụng nghiệp vụ với backend Java | 🌈        | 🌈         |
| Muốn tối ưu phía client              | 🛠️        | 🌈         |
| Muốn tạo UI động (on the fly)        | 🌈        | 🛠️         |
| Nhóm của tôi ưu tiên Java            | 🌈        | 🛠️         |
| Nhóm của tôi có nhiều dev frontend    | 🛠️        | 🌈         |
| Muốn tránh REST, GraphQL, v.v.        | 🌈        | 🛠️         |
| Xây dựng microservices               | 🛠️        | 🌈         |
| Tôi yêu thích an toàn kiểu dữ liệu (type safety) | 🌈        | 🌈         |
| Làm việc trên ứng dụng công khai cần SEO | 🛠️        | 🛠️         |

**Lưu ý về SEO (Dòng cuối cùng của bảng):**
Cả Vaadin Flow và Hilla đều không server-side rendered; cả hai đều là SPAs (Single Page Applications). Mặc dù các công cụ crawler hiện đại có thể hiểu nội dung, chúng được tối ưu cho trải nghiệm người dùng (UX) hơn là tối ưu hóa công cụ tìm kiếm (SEO). Đối với các ứng dụng web nghiệp vụ (không phải website công khai), SEO không phải lúc nào cũng là mối quan tâm chính. Metadata có thể được thêm riêng nếu cần.

**3. Khía cạnh kiến trúc:**

*   **Mô hình giao tiếp và quản lý trạng thái:**
    *   **Flow:** Sử dụng giao tiếp server-side. Server quản lý trạng thái UI, chỉ gửi các cập nhật tối thiểu đến client. Điều này đơn giản hóa phát triển cho các nhóm thuần Java, tự động liên kết dữ liệu và tăng cường bảo mật. Tuy nhiên, nó có thể gây ra thách thức về khả năng mở rộng (scalability) do các server có trạng thái (stateful servers). Mọi hành động của người dùng kích hoạt code Java trên backend, bạn không cần viết API.
    *   **Hilla:** Sử dụng giao tiếp client-side (REST/RPC). Client quản lý logic và trạng thái UI, yêu cầu dữ liệu từ một server không trạng thái (stateless server). Điều này mang lại sự linh hoạt cho client và khả năng mở rộng backend dễ dàng hơn. Bạn phải chủ động phơi bày các dịch vụ server-side thông qua các endpoint mà client (TypeScript) gọi một cách đồng bộ/bất đồng bộ.

*   **Những điểm tương đồng trên thực tế:**
    *   Các request của Hilla vẫn có thể hưởng lợi từ session cache trên server (trạng thái server).
    *   Flow có thể đẩy việc triển khai trạng thái component sang client, làm giảm tải cho server.
    *   Cả hai đều có tư duy lập trình theo "người dùng đơn lẻ" (single user perspective).
    *   Flow ngầm sử dụng "single user caching" cho dữ liệu server-side, trong khi Hilla là "shared cache". Tuy nhiên, thực tế thường cần kết hợp cả hai.
    *   Cả hai đều dành cho SPAs, nơi phát triển tập trung vào "thay đổi view" thay vì "trang" truyền thống.
    *   Hiện tại, quản lý trạng thái UI phản ứng dễ dàng hơn trong Hilla, nhưng điều này có thể thay đổi khi Flow nhận được tính năng "signals".

*   **Tóm quát về lựa chọn kiến trúc:**
    *   **Chọn Flow:** Để phát triển nhanh chóng với chuyên môn Java, ít tùy chỉnh phía client và UI được tạo động.
    *   **Chọn Hilla:** Cho các UI được thiết kế phong phú, tương tác cao, khi việc tối ưu phía client là quan trọng.

**4. Trải nghiệm nhà phát triển (Developer Ergonomics):**

*   **Gỡ lỗi (Debugging):**
    *   **Flow:** Mọi thứ chạy trên server. Bạn có thể gỡ lỗi toàn bộ vòng đời UI trực tiếp trong IDE Java của mình (đặt breakpoint JVM, kiểm tra beans, xem stack trace). Không cần phân biệt frontend và backend vì tất cả là một codebase. Gỡ lỗi đơn giản hơn khi mọi thứ nằm trong JVM.
    *   **Hilla:** Phân tách rõ ràng. Vấn đề phía client xử lý bằng công cụ devtools của trình duyệt. Vấn đề phía server dùng công cụ gỡ lỗi JVM. Thường dùng breakpoint trình duyệt để kiểm tra payload mạng và breakpoint JVM trên server. Nhờ các kiểu TypeScript được tạo tự động, ít gặp lỗi "undefined is not a function".

*   **Triển khai nóng và tải lại trực tiếp (Hot deployment and live reload):**
    *   Cả hai framework đều hỗ trợ việc chỉnh sửa code và xem thay đổi được triển khai ngay lập tức, nhờ vào tooling của nền tảng Vaadin (HotSwapAgent cho JVM và Vite cho frontend trong chế độ phát triển).

**5. Tóm tắt cuối cùng:**

*   **Vaadin Flow:**
    *   Framework web full-stack cho phép xây dựng SPAs hoàn toàn bằng Java.
    *   Cung cấp API component dựa trên Java và tự động hóa giao tiếp client-server.
    *   Ứng dụng chạy trên server, truy cập trực tiếp vào dữ liệu.
    *   **Tính năng chính:** Hệ thống thiết kế tùy chỉnh với hơn 50 component UI, định tuyến và xử lý form tích hợp, hỗ trợ quốc tế hóa, dependency injection (tương thích Spring và CDI).
    *   **Lợi ích:** Đơn giản hóa phát triển web, tập trung vào logic nghiệp vụ.

*   **Vaadin Hilla:**
    *   Framework web full-stack Java được thiết kế để xây dựng ứng dụng client-side.
    *   Kết hợp mô hình component dựa trên TypeScript với template khai báo, phản ứng và render DOM hiệu quả.
    *   **Tính năng chính:** Hệ thống thiết kế tùy chỉnh với hơn 50 component UI, định tuyến và chia tách code tích hợp, giao tiếp không đồng bộ, an toàn kiểu dữ liệu với backend Java. Tự động tạo định nghĩa kiểu chung từ các lớp server.
    *   **Lợi ích:** Đảm bảo thông tin kiểu nhất quán giữa server và client, giúp bắt lỗi API sớm hơn (lúc biên dịch). Thích hợp cho các nhóm ưa chuộng tooling frontend hiện đại.
