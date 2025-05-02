# 1 Fetch API vs. Axios vs. Alova: Lựa chọn HTTP cho năm 2025

Đây là bản tóm tắt so sánh giữa **Fetch API**, **Axios** và **Alova**, giúp bạn quyết định nên sử dụng trình khách HTTP (HTTP client) nào cho dự án của mình vào năm 2025, dựa trên các tính năng, trường hợp sử dụng và hệ sinh thái của chúng.

## So sánh từng tính năng

| Tính năng             | Fetch API                              | Axios                                       | Alova                                          |
| :-------------------- | :------------------------------------- | :------------------------------------------ | :--------------------------------------------- |
| **Dễ sử dụng**        | Trung bình (cần xử lý thủ công nhiều) | Cao (cú pháp thân thiện)                   | Trung bình (cần học các mẫu mới)             |
| **Hiệu suất**         | Cao (nhẹ, gốc)                         | Trung bình (kích thước lớn hơn một chút)   | Cao (tối ưu hóa cho caching & batch request) |
| **Xử lý JSON**        | Thủ công (`.json()`)                    | Tự động                                     | Tự động                                        |
| **Hủy yêu cầu**      | `AbortController` (thủ công)           | Hỗ trợ `AbortController`                    | Tích hợp sẵn (thông qua adapter)             |
| **Interceptors**      | Không                                  | Có                                          | Có (thông qua middleware/hook)               |
| **Xử lý Timeout**     | Không (thủ công với `AbortController`) | Có (tích hợp)                              | Có (tích hợp)                                 |
| **Caching dữ liệu**   | Không                                  | Không (cần thư viện bên thứ ba)             | Có (tích hợp, đa cấp)                         |
| **Cơ chế thử lại**    | Không                                  | Có (thông qua thư viện adapter hoặc tùy chỉnh) | Có (tích hợp)                                 |
| **Xử lý lỗi**         | Yêu cầu xử lý thủ công (`response.ok`) | Tự động reject cho mã trạng thái không phải 2xx | Tích hợp phục hồi lỗi, quản lý trạng thái lỗi |
| **Hỗ trợ trình duyệt** | Mọi trình duyệt hiện đại                | Mọi trình duyệt hiện đại                    | Mọi trình duyệt hiện đại                        |
| **Hỗ trợ Node.js**   | Có (từ v17.5+)                         | Có                                          | Có                                             |

*_(Lưu ý: Một số mục trong bảng trên đã được điều chỉnh so với bài viết gốc để phản ánh chính xác hơn, ví dụ: Axios hỗ trợ AbortController, Alova hỗ trợ Node.js)._*

## Trường hợp sử dụng và Kịch bản tốt nhất

Việc chọn trình khách HTTP phù hợp phụ thuộc vào độ phức tạp của dự án, các dependency và các cân nhắc về hiệu suất.

### Khi nào nên dùng Fetch API

*   **Phù hợp cho các dự án nhẹ và yêu cầu đơn giản:** Lý tưởng để xử lý các yêu cầu HTTP cơ bản mà không cần thêm dependency.
*   **Khi làm việc trong môi trường hạn chế thư viện bên thứ ba:** Là lựa chọn khả thi khi không được phép sử dụng các gói của bên thứ ba như Axios hoặc Alova.
*   **Khi ưu tiên ít dependency nhất:** Hoàn hảo cho các dự án cần giữ số lượng dependency ở mức thấp (ứng dụng nhỏ, nhẹ, trang web tĩnh).

### Khi nào nên dùng Axios

*   **Lý tưởng cho các ứng dụng nặng về backend hoặc API phức tạp:** Lựa chọn vững chắc khi cần nhiều lệnh gọi API, xử lý lỗi mạnh mẽ và quản lý yêu cầu hiệu quả.
*   **Khi cần xử lý JSON tự động, interceptors và xử lý lỗi mạnh mẽ:** Đơn giản hóa việc làm việc với JSON, cung cấp interceptors tích hợp và xử lý lỗi HTTP tự động tốt hơn Fetch.
*   **Hữu ích khi làm việc với Node.js trong các ứng dụng full-stack:** Hoạt động trên cả trình duyệt và Node.js, phù hợp cho các ứng dụng cần sự thống nhất giữa frontend và backend.

### Khi nào nên dùng Alova

*   **Khi làm việc với các ứng dụng nặng về frontend (React, Vue, Svelte):** Tích hợp tốt với các framework frontend và quản lý trạng thái, tuyệt vời cho các SPA cần fetch dữ liệu mượt mà, phân trang, cập nhật.
*   **Tốt nhất cho các dự án yêu cầu caching tối ưu và đồng bộ hóa dữ liệu:** Được thiết kế để tối ưu hóa hiệu suất và caching, phù hợp cho các ứng dụng cần giảm thiểu yêu cầu mạng dư thừa.
*   **Khi tối ưu hóa hiệu suất và giảm tải mạng là ưu tiên:** Các cơ chế caching thông minh giúp giảm đáng kể tần suất gọi API, cải thiện hiệu suất, đặc biệt hữu ích cho ứng dụng di động hoặc PWA.

## Cộng đồng và Hệ sinh thái

### Hệ sinh thái và Tích hợp

*   **Fetch API:** Được hỗ trợ rộng rãi, nhưng thường cần thư viện bổ sung cho các tính năng nâng cao (caching, timeout), làm tăng nỗ lực phát triển.
*   **Axios:** Hệ sinh thái lâu đời, nhiều plugin và tiện ích mở rộng, dễ tích hợp với nhiều hệ thống khác nhau.
*   **Alova:** Hoạt động liền mạch với các thư viện quản lý trạng thái hiện đại (TanStack Query, Zustand, Pinia), hấp dẫn cho việc tối ưu hóa fetch dữ liệu frontend, có thể thay thế các thư viện quản lý trạng thái fetch dữ liệu riêng biệt.

## Kết luận

Việc lựa chọn giữa Fetch API, Axios và Alova phụ thuộc vào nhu cầu và ưu tiên của dự án:

*   **Fetch API:** Tốt nhất cho các ứng dụng **nhẹ**, yêu cầu **ít dependency** nhất.
*   **Axios:** Lựa chọn **mạnh mẽ**, **đã được kiểm chứng** cho ứng dụng **full-stack**, môi trường **backend**, hoặc khi cần các **tính năng tiện lợi** (interceptors, xử lý lỗi tự động) mà **không cần thêm thư viện quản lý trạng thái phức tạp**.
*   **Alova:** Lựa chọn **hiện đại**, **tuyệt vời** để **tối ưu hóa** việc fetch dữ liệu và **caching** trong các ứng dụng tập trung vào **frontend** (đặc biệt là SPA, PWA), nơi **hiệu suất** và **trải nghiệm người dùng** là tối quan trọng.

# 2 Tóm tắt: So sánh hiệu năng SSR - Fastify + React/Vue vs. Next.js/Nuxt (Tháng 4/2025)

Bài viết này trình bày kết quả so sánh hiệu năng Server-Side Rendering (SSR) giữa việc chạy ứng dụng React/Vue trên **Fastify** (sử dụng `@fastify/react` và `@fastify/vue`) so với việc chạy chúng trong các metaframework tương ứng là **Next.js** và **Nuxt**.

## Ý chính

1.  **Thiết lập Benchmark:**
    *   Tạo các ứng dụng benchmark đơn giản (mã "spiral") cho mỗi cấu hình:
        *   Next.js (sử dụng `create-next-app`, App Router, `export const dynamic = 'force-dynamic'`).
        *   Nuxt (sử dụng `create-nuxt-app`, component `<Head>`, `<Meta>`, `<Style>`).
        *   Fastify + React (sử dụng `@fastify/react` với `@fastify/vite`).
        *   Fastify + Vue (sử dụng `@fastify/vue` với `@fastify/vite`).
    *   Mục tiêu là so sánh hiệu năng của các **thiết lập mặc định (vanilla setup)**, không phải các cấu hình đã được tối ưu hóa đặc biệt.

2.  **Kết quả Hiệu năng (Yêu cầu/giây):**
    *   `@fastify/vue`: **717 req/s**
    *   `Nuxt`: 561 req/s
    *   `@fastify/react`: **347 req/s**
    *   `Next.js`: **49 req/s**

3.  **Phát hiện Nổi bật:**
    *   **Fastify + React nhanh hơn khoảng 7 lần so với Next.js** trong bài benchmark này.
    *   Fastify + Vue cũng nhanh hơn Nuxt.
    *   Tác giả bày tỏ sự ngạc nhiên về mức chênh lệch hiệu năng lớn với Next.js và đặt câu hỏi về nguồn gốc của overhead (có thể do App Router?).

4.  **Lưu ý Quan trọng (Caveats):**
    *   **So sánh không hoàn toàn công bằng:**
        *   `@fastify/react` và `@fastify/vue` được mô tả là **cực kỳ tối giản**, cung cấp những gì cốt lõi nhất cho SSR và chuyển đổi sang SPA.
        *   Next.js và Nuxt là các **framework lớn ("behemoths", "Swiss Army knives")**, cung cấp hệ sinh thái đầy đủ, xử lý nhiều trường hợp biên và tích hợp sẵn nhiều tính năng ("batteries included").
    *   **Sự đánh đổi:** Việc sử dụng Next.js/Nuxt mang lại sự tiện lợi và một "con đường" (hệ sinh thái) được định sẵn, nhưng đi kèm với chi phí hiệu năng (overhead) tiềm ẩn. Các giải pháp Fastify mang lại hiệu năng cao hơn cho chức năng SSR cốt lõi nhưng yêu cầu tự xây dựng nhiều thứ hơn.

## Kết luận Tóm tắt

Trong bài kiểm tra hiệu năng SSR cụ thể này, việc sử dụng React và Vue trực tiếp với Fastify cho thấy hiệu năng vượt trội đáng kể so với việc sử dụng chúng trong Next.js và Nuxt, đặc biệt là trường hợp Fastify + React so với Next.js (nhanh hơn ~7 lần). Tuy nhiên, cần lưu ý rằng Next.js và Nuxt cung cấp một bộ tính năng phong phú hơn nhiều và một trải nghiệm phát triển tích hợp sẵn, điều này có thể lý giải cho phần overhead về hiệu năng. Lựa chọn giữa chúng phụ thuộc vào việc ưu tiên hiệu năng tối đa cho phần cốt lõi SSR hay cần một hệ sinh thái framework toàn diện.
*(Dựa trên bài viết ngày 9 tháng 4 năm 2025)*