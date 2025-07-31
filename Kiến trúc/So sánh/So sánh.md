

### So sánh Kiến trúc Đảo với các mô hình render khác

Để hiểu rõ lợi ích của Kiến trúc Đảo, chúng ta cần so sánh nó với các mô hình render truyền thống:

| Đặc điểm / Mô hình | **Client-Side Rendering (CSR)** | **Server-Side Rendering (SSR)** | **Static Site Generation (SSG)** | **Kiến trúc Đảo (Islands Architecture)** |
| :------------------ | :------------------------------ | :------------------------------ | :--------------------------------- | :--------------------------------------- |
| **Nội dung ban đầu (HTML)** | Thường là HTML rỗng/tối thiểu (shell). | HTML đầy đủ, đã render từ server. | HTML đầy đủ, đã render sẵn tại thời điểm build. | HTML đầy đủ, đã render từ server (hoặc build). |
| **JavaScript (JS) ban đầu** | **Lớn**, cần để render và làm tương tác toàn bộ ứng dụng. | **Lớn**, cần để "thủy hóa" (rehydrate) toàn bộ trang. | **Rất ít hoặc không có**, nếu không có tương tác. | **Rất ít và phân mảnh**, chỉ cho các "đảo" tương tác. |
| **Thời gian hiển thị nội dung đầu tiên (FCP)** | Chậm (phải tải JS, render). | Nhanh (HTML có sẵn). | Rất nhanh (HTML có sẵn). | Rất nhanh (HTML có sẵn). |
| **Thời gian tương tác (TTI)** | Có thể rất chậm (chờ JS tải, parse, execute, hydrate toàn bộ). | Có thể chậm (chờ JS tải, parse, execute, hydrate toàn bộ). | Rất nhanh (ít JS). | Nhanh (chỉ các đảo nhỏ hydrate độc lập). |
| **SEO**             | Kém (cần JS để nội dung hiện ra). | Tốt (nội dung có sẵn trong HTML). | Rất tốt (nội dung có sẵn trong HTML). | Tốt (nội dung có sẵn trong HTML). |
| **Tính tương tác**   | Toàn bộ trang là "ứng dụng" tương tác. | Toàn bộ trang được thủy hóa để tương tác. | Hạn chế, chỉ có thể thêm tương tác nhỏ bằng JS độc lập. | Các "đảo" tương tác nhỏ, độc lập, phần còn lại là tĩnh. |
| **Chi phí "thủy hóa"** | Cao (thủy hóa toàn bộ ứng dụng). | Cao (thủy hóa toàn bộ trang). | Rất thấp (nếu có). | Rất thấp (chỉ thủy hóa các đảo nhỏ). |
| **Phù hợp với**     | Ứng dụng web động, dashboards. | Website nhiều nội dung, cần SEO (blog, tin tức). | Website tĩnh, ít thay đổi, blog, tài liệu. | Website chủ yếu tĩnh nhưng cần một số khu vực tương tác (blog có comment/share, trang sản phẩm có carousel/filter). |

---

### Lợi ích của Kiến trúc Đảo (Benefits)

Kiến trúc Đảo kết hợp các ý tưởng từ SSR, SSG và partial hydration để mang lại những lợi ích đáng kể:

1.  **Hiệu suất vượt trội:**
    *   **Giảm đáng kể lượng JavaScript:** Đây là lợi ích lớn nhất. Thay vì gửi toàn bộ JS để render lại DOM ảo và thủy hóa toàn bộ trang, Kiến trúc Đảo chỉ gửi code JS cần thiết cho các component tương tác cụ thể (các "đảo"). Điều này làm cho tổng kích thước JS được tải xuống và xử lý bởi trình duyệt nhỏ hơn rất nhiều. Ví dụ, Astro (một framework áp dụng Kiến trúc Đảo) đã cho thấy giảm 83% lượng JS so với các trang tài liệu của Next.js và Nuxt.js.
    *   **Tải trang nhanh hơn (Faster Page Loads):** Với ít JS hơn, trang web hiển thị nội dung nhanh chóng hơn.
    *   **Thời gian tương tác (TTI) thấp hơn:** Người dùng có thể tương tác với trang nhanh hơn vì không phải chờ toàn bộ JS của trang được tải và thủy hóa. Chỉ các "đảo" nhỏ, cần thiết mới được kích hoạt tương tác.

2.  **Tối ưu SEO:**
    *   Vì phần lớn nội dung tĩnh của trang được render sẵn trên server và gửi dưới dạng HTML thuần túy, các công cụ tìm kiếm có thể dễ dàng thu thập và đánh chỉ mục (index) nội dung, mang lại lợi ích SEO vượt trội.

3.  **Ưu tiên nội dung quan trọng:**
    *   Nội dung cốt lõi của trang (ví dụ: bài viết blog, mô tả sản phẩm) hiển thị gần như ngay lập tức cho người dùng. Các chức năng tương tác phụ trợ (như nút chia sẻ, carousel hình ảnh) sẽ được tải và kích hoạt dần dần sau đó, không làm cản trở trải nghiệm đọc/xem nội dung chính.

4.  **Cải thiện khả năng truy cập (Accessibility):**
    *   Việc sử dụng HTML tĩnh chuẩn và các liên kết cơ bản để điều hướng giữa các trang giúp cải thiện khả năng truy cập của website, đảm bảo trang vẫn hoạt động tốt ngay cả khi JS bị tắt hoặc gặp lỗi.

5.  **Dựa trên Component:**
    *   Kiến trúc Đảo kế thừa tất cả các ưu điểm của kiến trúc dựa trên component, bao gồm khả năng tái sử dụng component, dễ dàng bảo trì code, và phân chia trách nhiệm rõ ràng.
    *   Các "đảo" hoạt động độc lập, giúp cô lập lỗi: một vấn đề trong một "đảo" tương tác sẽ ít có khả năng ảnh hưởng đến hiệu suất hoặc chức năng của các phần khác trên trang.

**Tóm lại:**

Kiến trúc Đảo là một cách tiếp cận mạnh mẽ để xây dựng các trang web nhanh, hiệu quả và thân thiện với SEO bằng cách chỉ gửi JavaScript cần thiết đến trình duyệt cho các thành phần tương tác, trong khi phần lớn nội dung còn lại là HTML tĩnh. Nó đặc biệt phù hợp cho các trang web có nhiều nội dung nhưng cần một vài khu vực tương tác cụ thể, như blog, trang tin tức, hoặc trang sản phẩm trong e-commerce. Mặc dù còn tương đối mới và yêu cầu sự hỗ trợ từ framework, nhưng tiềm năng tối ưu hiệu suất của nó là rất lớn.