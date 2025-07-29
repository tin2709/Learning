

# Tóm tắt và Giải thích: "Top 5 lựa chọn thay thế Next.js cho Lập trình viên React" (từ LogRocket)

Bài viết từ LogRocket phân tích lý do tại sao các lập trình viên đang tìm kiếm các lựa chọn thay thế Next.js, chủ yếu do sự phức tạp ngày càng tăng, các tính năng quá chuyên biệt (opinionated), và lo ngại về sự phụ thuộc vào Vercel (vendor lock-in). Dưới đây là 5 lựa chọn thay thế hàng đầu dành cho những ai muốn tiếp tục sử dụng React.

## Bảng tổng quan các lựa chọn thay thế

| Framework | Phù hợp nhất cho | Hỗ trợ React | Hỗ trợ SSR | Routing | Tải dữ liệu | An toàn kiểu (Type Safety) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Remix** | Ứng dụng full-stack với cơ chế form/dữ liệu tích hợp | Đầy đủ | Có | Dựa trên file + `loaders`/`actions` | `Loaders` + `actions` (tích hợp) | Có |
| **Astro** | Trang web tĩnh hoặc hybrid, nhiều nội dung | Một phần | Có (mới hơn) | Dựa trên file với "islands" | `fetch` trong component Astro | Có |
| **TanStack Start** | Ứng dụng React full-stack, an toàn kiểu hoàn toàn | Đầy đủ | Có | Dựa trên file qua TanStack Router | Server functions + typed loaders | **Có (full-stack)** |
| **Vike** | Toàn quyền kiểm soát SSR/SSG với ít trừu tượng | Đầy đủ | Có | Dựa trên quy ước (`+Page.tsx`) | Hook server tùy chỉnh (`onBeforeRender`) | Có |
| **Vite + React Router**| Ứng dụng React phía client nhẹ nhàng | Đầy đủ | Không (cần cài đặt thủ công)| Thủ công qua React Router | `loaders` của React Router | Có |

---

## Phân tích chi tiết từng lựa chọn

### 1. Remix

Remix là một trong những đối thủ mạnh nhất của Next.js, được xây dựng dựa trên các tính năng web gốc (forms, caching, HTTP) thay vì tạo ra các "phép thuật" trừu tượng.

*   **Phù hợp nhất cho**: Ứng dụng web full-stack phức tạp.
*   **Ưu điểm**:
    *   **Routing thông minh**: Mỗi route có thể tự định nghĩa hàm `loader()` để lấy dữ liệu và `action()` để xử lý mutations (gửi form). Tất cả đều chạy trên server.
    *   **SSR mặc định**: SSR là mặc định và được tích hợp chặt chẽ.
    *   **Caching hiệu quả**: Sử dụng các header HTTP chuẩn (`Cache-Control`, `ETags`).
*   **Nhược điểm**:
    *   Hệ sinh thái nhỏ hơn Next.js.
    *   Tư tưởng "thuận theo web" có thể đòi hỏi nhiều công sức hơn nếu bạn đã quen với các framework trừu tượng hóa cao.

### 2. Astro

Astro được thiết kế cho các trang web nặng về nội dung (blog, trang tài liệu, landing page). Sức mạnh cốt lõi của nó là **"Island Architecture"**.

*   **Phù hợp nhất cho**: Các trang web tĩnh, nặng về nội dung, ưu tiên tốc độ tải trang.
*   **Ưu điểm**:
    *   **Partial Hydration**: Mặc định gửi về HTML tĩnh và chỉ "thủy hóa" (hydrate) các component tương tác cần thiết (`client:*` directive), giúp dung lượng JavaScript cực nhỏ.
    *   **Linh hoạt về framework**: Có thể dùng React, Vue, Svelte... trong cùng một dự án.
    *   **Hỗ trợ Markdown/MDX xuất sắc**.
*   **Nhược điểm**:
    *   Không phải là một framework React hoàn chỉnh, có thể hạn chế nếu xây dựng ứng dụng phức tạp, tương tác cao (SPA, dashboard).
    *   Hỗ trợ SSR còn khá mới.

### 3. TanStack Start

Đây là một framework full-stack mới từ đội ngũ đã tạo ra TanStack Query, Table... Mục tiêu là xây dựng ứng dụng React nhanh, **an toàn về kiểu (type-safe)** từ đầu đến cuối.

*   **Phù hợp nhất cho**: Ứng dụng nặng về dữ liệu, ưu tiên tuyệt đối về type-safety.
*   **Ưu điểm**:
    *   **An toàn kiểu từ front-end đến back-end**: Tích hợp chặt chẽ với TanStack Router.
    *   **SSR và Streaming**: Hỗ trợ sẵn.
    *   **Ít boilerplate**: Cảm giác giống như xây dựng một ứng dụng React thuần túy, không bị ép vào một cấu trúc thư mục cứng nhắc.
*   **Nhược điểm**:
    *   **Còn rất mới (beta)**: API có thể thay đổi và tài liệu chưa hoàn chỉnh.

### 4. Vike (trước đây là vite-ssr)

Vike là một meta-framework nhẹ, xây dựng trên Vite, cho phép bạn **toàn quyền kiểm soát** mọi thứ. Nó không áp đặt cấu trúc hay các cơ chế "hộp đen".

*   **Phù hợp nhất cho**: Những ai muốn toàn quyền kiểm soát SSR, SSG và kiến trúc ứng dụng.
*   **Ưu điểm**:
    *   **Không áp đặt (Unopinionated)**: Bạn tự quyết định cách tổ chức code, công cụ tải dữ liệu, caching...
    *   **Kiểm soát SSR/SSG trên từng route**: Rất linh hoạt.
    *   **Minh bạch**: Không có các hành vi ẩn.
*   **Nhược điểm**:
    *   **Không có lớp dữ liệu tích hợp sẵn**: Bạn phải tự xây dựng logic fetching, caching.
    *   **Rào cản gia nhập cao hơn**: Yêu cầu người dùng phải có kiến thức vững về SSR, routing...

### 5. Vite + React Router

Đây không phải là một framework chính thức mà là một "combo" hiện đại. Vite cung cấp công cụ build siêu nhanh, và React Router (từ v6.4) đã hỗ trợ tải dữ liệu ở cấp độ route.

*   **Phù hợp nhất cho**: Các ứng dụng phía client (SPA) nhẹ nhàng, nhanh chóng.
*   **Ưu điểm**:
    *   **Công cụ build hiện đại (Vite)**: Dev server khởi động tức thì, HMR siêu nhanh.
    *   **Routing có nhận biết dữ liệu**: Dùng `loader` của React Router để fetch dữ liệu trước khi render route.
*   **Nhược điểm**:
    *   **Không có routing dựa trên file**: Phải định nghĩa route thủ công trong code.
    *   **Không có SSR tích hợp sẵn**: Phải tự cài đặt nếu cần.

---

## Kết luận: Nên chọn lựa chọn nào?

Bài viết đưa ra lời khuyên rất thực tế:

> *   **Dùng Remix** nếu bạn muốn một framework React full-stack mạnh mẽ với cơ chế xử lý dữ liệu thông minh và ít trừu tượng.
> *   **Dùng Astro** nếu bạn xây dựng một trang web nặng về nội dung (blog, marketing) và ưu tiên tốc độ tối đa.
> *   **Dùng TanStack Start** nếu bạn đang xây dựng một ứng dụng nặng về dữ liệu và cực kỳ quan tâm đến an toàn kiểu (type safety).
> *   **Dùng Vike** nếu bạn muốn toàn quyền kiểm soát mọi thứ (routing, SSR, data loading) mà không bị framework áp đặt.
> *   **Dùng Vite + React Router** nếu bạn muốn một ứng dụng SPA nhẹ, nhanh, với công cụ hiện đại mà không cần các tính năng full-stack phức tạp.



# 2 Phát Trực Tuyến Video Thời Gian Thực Với Next.js: HLS.js và Các Giải Pháp Thay Thế

Bài viết này cung cấp hướng dẫn chi tiết về cách triển khai tính năng phát trực tuyến video thời gian thực trong ứng dụng Next.js. Chúng ta sẽ khám phá HLS.js, một thư viện JavaScript phổ biến cho phép phát trực tuyến HTTP Live Streaming (HLS) trực tiếp trên trình duyệt, cùng với các tính năng nâng cao và so sánh với các giải pháp thay thế mã nguồn mở khác.

## HLS.js là gì?
HLS.js là một thư viện JavaScript mã nguồn mở cho phép trình duyệt phát nội dung HTTP Live Streaming (HLS). HLS là một phương pháp phát video phổ biến do Apple tạo ra, giúp phân phối video trực tiếp và theo yêu cầu mượt mà qua các máy chủ web tiêu chuẩn.

HLS.js xử lý các tệp kê khai HLS (.m3u8), phân tách luồng video và âm thanh, sau đó cung cấp chúng cho phần tử HTML5 `<video>` tiêu chuẩn. Cấu trúc này giúp thích ứng với các điều kiện mạng thay đổi, đảm bảo chất lượng video tốt nhất với ít bộ đệm.

**Các tính năng chính của HLS.js:**
*   **Phát trực tuyến tốc độ bit thích ứng (ABR):** Tự động thay đổi chất lượng video dựa trên tốc độ internet và hiệu suất thiết bị.
*   **Hỗ trợ phát trực tiếp và theo yêu cầu:** Xử lý cả phát sóng trực tiếp và nội dung video theo yêu cầu (VOD).
*   **Hỗ trợ fMP4 và MPEG-2 TS:** Tương thích với các định dạng phân đoạn HLS phổ biến.
*   **Luồng chỉ âm thanh:** Có thể quản lý các luồng chỉ bao gồm âm thanh.
*   **Xử lý sự kiện mở rộng:** Cung cấp nhiều sự kiện để theo dõi quá trình phát trực tuyến, xử lý lỗi, giám sát bộ đệm.
*   **Hỗ trợ HLS độ trễ thấp (LL-HLS):** Giảm độ trễ cho các sự kiện trực tiếp.
*   **Tối ưu hóa hiệu suất:** Tinh chỉnh bộ đệm, tải trước phân đoạn và quản lý yêu cầu mạng.

## Yêu cầu tiên quyết
Trước khi bắt đầu, hãy đảm bảo bạn có:
*   Node.js (v18+) đã cài đặt.
*   Dự án Next.js (v14+) đã được thiết lập (với App Router).
*   Kiến thức cơ bản về React Hooks và JavaScript.
*   Hiểu biết cơ bản về HTML5 Video.

## Thiết lập dự án
1.  **Tạo dự án Next.js mới:**
    ```bash
    npx create-next-app@latest next-video-streaming
    ```
    *Khi được nhắc, hãy chọn: TypeScript: Yes, ESLint: Yes, Tailwind CSS: Yes, App Router: Yes, Import alias: Yes.*
2.  **Di chuyển vào thư mục dự án:**
    ```bash
    cd next-video-streaming
    ```
3.  **Cài đặt `hls.js`:**
    ```bash
    npm install hls.js
    npm install --save-dev @types/hls.js # (Nếu dùng TypeScript)
    ```

## Triển khai phát trực tuyến
Để bắt đầu phát sóng trực tiếp bằng HLS, bạn cần tạo và quản lý một danh sách phát HLS (playlist) liên tục cập nhật các đoạn video mới.

### Thiết lập thành phần Live Stream
Tạo một thành phần `LiveStream` mới (ví dụ: `components/live-stream.tsx`). Thành phần này nhận một URL playlist HLS (.m3u8) và sử dụng lớp `Hls` để tải luồng vào phần tử `<video>` của HTML5.

*Cấu hình quan trọng như `liveSyncDurationCount` và `liveMaxLatencyDurationCount` giúp giữ luồng gần thời gian thực, giảm độ trễ trong khi vẫn đảm bảo phát lại mượt mà.*

Sau khi tạo thành phần, bạn có thể nhập nó vào tệp `page.tsx` của mình:
```typescript
'use client';
import LiveStream from "@/components/live-stream";

export default function Home() {
  return (
    <div className="flex flex-col justify-center items-center h-screen">
      <h1 className="text-2xl font-bold mb-4">Live Stream</h1>
      <LiveStream playlistUrl="https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8" />
    </div>
  );
}
```

### Quản lý trạng thái luồng bằng sự kiện HLS.js
HLS.js cung cấp một hệ thống sự kiện phong phú giúp bạn quản lý và kiểm soát quá trình phát trực tuyến. Bạn có thể lắng nghe các sự kiện như:
*   `Hls.Events.MANIFEST_PARSED`: Xảy ra khi danh sách phát chính được tải và phân tích cú pháp.
*   `Hls.Events.LEVEL_LOADED`: Kích hoạt khi một danh sách phát mức chất lượng được tải.
*   `Hls.Events.FRAG_LOADED`: Xảy ra khi các đoạn video tải thành công.
*   `Hls.Events.ERROR`: Xử lý các lỗi phát trực tuyến khác nhau.
*   `Hls.Events.BUFFER_APPENDED`: Chỉ ra khi dữ liệu video được thêm vào bộ đệm.

*Việc tích hợp các sự kiện này vào `useEffect` của thành phần `LiveStream` giúp cập nhật trạng thái `isLive` (khi manifest được phân tích cú pháp) và hiển thị thông báo lỗi kịp thời (với `Hls.Events.ERROR`).*

## Các tính năng nâng cao của HLS.js

### Xử lý nhiều mức chất lượng cho phát trực tuyến tốc độ bit thích ứng (ABR)
ABR là tính năng quan trọng để cung cấp trải nghiệm video mượt mà bất kể kết nối internet hay thiết bị sử dụng. HLS.js tự động điều chỉnh chất lượng video dựa trên băng thông mạng, trạng thái bộ đệm và khả năng thiết bị.

Bạn có thể cho phép người dùng tự chọn mức chất lượng video thông qua giao diện người dùng, sử dụng các sự kiện như `Hls.Events.MANIFEST_PARSED` để lấy danh sách các mức chất lượng (`hls.levels`) và hàm `changeQuality` để thiết lập `hls.currentLevel`.

### Triển khai xác thực dựa trên mã thông báo (Token-based authentication) cho luồng bảo mật
Bảo vệ các luồng HLS là rất quan trọng đối với nội dung cao cấp hoặc phát sóng riêng tư. HLS.js cho phép bạn thêm mã thông báo xác thực vào các yêu cầu playlist và tải xuống phân đoạn thông qua callback `xhrSetup`.

Bạn sẽ cần một API backend để xác thực người dùng, cung cấp mã thông báo. HLS.js sau đó sẽ thêm mã thông báo này vào các tiêu đề yêu cầu HTTP (`Authorization`, `X-Custom-Auth`, v.v.). Lắng nghe sự kiện `Hls.Events.ERROR` giúp bạn xử lý các lỗi xác thực (ví dụ: mã thông báo không hợp lệ) và hiển thị thông báo lỗi rõ ràng.

### Tối ưu hóa hiệu suất với cấu hình HLS.js
HLS.js cung cấp nhiều tùy chọn cấu hình để cải thiện hiệu suất:
*   `liveSyncDurationCount`: Giữ người chơi chậm hơn một số phân đoạn so với phát sóng trực tiếp để cân bằng độ trễ và ổn định.
*   `liveMaxLatencyDurationCount`: Bỏ qua phân đoạn nếu độ trễ vượt quá ngưỡng nhất định.
*   `lowLatencyMode`: Kích hoạt các tính năng độ trễ thấp (<3 giây).
*   `maxBufferLength`, `maxMaxBufferLength`: Giới hạn kích thước bộ đệm để giảm sử dụng bộ nhớ.
*   `backBufferLength`: Giữ nội dung đã qua để tính năng DVR.
*   `abrEwmaFastLive`, `abrEwmaSlowLive`: Điều chỉnh nhanh chóng/chậm rãi theo thay đổi mạng.
*   `enableWorker`: Sử dụng Web Worker để xử lý, cải thiện hiệu suất luồng chính.
*   `startLevel`: Cho phép HLS.js tự động chọn chất lượng tốt nhất.

## Các lựa chọn thay thế HLS.js
Ngoài HLS.js, có một số lựa chọn mã nguồn mở khác, mỗi lựa chọn có điểm mạnh riêng phù hợp với các trường hợp sử dụng khác nhau.

### 1. Video.js
*   **Mô tả:** Một công cụ mã nguồn mở phổ biến để phát video HTML5, hoạt động nhất quán trên nhiều trình duyệt.
*   **Tính năng chính:** Tương thích đa trình duyệt, UI tùy chỉnh, kiến trúc plugin (quan trọng cho HLS qua `videojs-contrib-hls`), hỗ trợ phát trực tuyến thích ứng (qua plugin), trợ năng.
*   **Ưu điểm:** Cộng đồng lớn, hệ sinh thái plugin phong phú, tài liệu rõ ràng.
*   **Nhược điểm:** Cần plugin phụ cho HLS, kích thước lớn hơn HLS.js, tối ưu hóa nâng cao có thể phức tạp.
*   **Cách dùng cơ bản:** `npm install video.js`, sau đó nhúng thành phần `<video>` và khởi tạo `videojs`.

### 2. Stream API (Video)
*   **Mô tả:** Cung cấp các API quản lý việc phát trực tuyến và video theo yêu cầu, giảm gánh nặng kỹ thuật cho nhà phát triển.
*   **Tính năng chính:** Cơ sở hạ tầng phát trực tuyến được quản lý (tải lên, chuyển mã, lưu trữ, CDN), hỗ trợ trực tiếp và theo yêu cầu, tùy chọn độ trễ thấp, khả năng mở rộng.
*   **Ưu điểm:** Đơn giản hóa backend phức tạp, không cần quản lý FFmpeg/CDN, hiệu suất ổn định cho lượng lớn khán giả.
*   **Nhược điểm:** Dịch vụ trả phí (dựa trên sử dụng), ít kiểm soát hơn FFmpeg, phụ thuộc vào nhà cung cấp dịch vụ.
*   **Cách dùng cơ bản:** `npm install @stream-io/video-react-sdk`, sử dụng `StreamVideoClient` và `StreamCall`.

### 3. Daily.co
*   **Mô tả:** API và SDK dễ sử dụng để thêm cuộc gọi video/âm thanh thời gian thực (WebRTC) vào ứng dụng web/di động.
*   **Tính năng chính:** Tập trung vào WebRTC (độ trễ thấp), SDK đa nền tảng, phát trực tiếp tương tác (lên đến 100k người tham gia), ghi hình, chia sẻ màn hình, tính năng sẵn sàng AI, mạng lưới toàn cầu.
*   **Ưu điểm:** Thời gian phản hồi rất nhanh, đơn giản hóa WebRTC, chất lượng video tốt, phù hợp cho ứng dụng tương tác trực tiếp.
*   **Nhược điểm:** Dịch vụ trả phí (freemium), tối ưu cho giao tiếp thời gian thực hơn là VOD quy mô lớn, hệ thống chính do Daily.co kiểm soát.
*   **Cách dùng cơ bản:** `npm i @daily-co/daily-react`, sử dụng `DailyProvider`.

### 4. Node.js/FFmpeg
*   **Mô tả:** Kết hợp Node.js (backend) và FFmpeg (công cụ xử lý đa phương tiện) để tùy chỉnh hoàn toàn quá trình phát trực tuyến.
*   **Tính năng chính:** Kiểm soát toàn bộ quá trình (từ thu thập đến phân phối), tương thích nhiều định dạng, chuyển mã/đóng gói/phân đoạn, xử lý luồng trực tiếp.
*   **Ưu điểm:** Tùy chỉnh mọi khía cạnh, chi phí vận hành có thể rẻ hơn (sau thiết lập ban đầu), toàn quyền sở hữu.
*   **Nhược điểm:** Yêu cầu kiến thức chuyên sâu, triển khai phức tạp, chịu trách nhiệm về khả năng mở rộng và giám sát.
*   **Cách dùng cơ bản:** Sử dụng `child_process.spawn` để gọi lệnh FFmpeg từ Node.js.

### 5. Dolby OptiView
*   **Mô tả:** Giải pháp phát trực tuyến video chất lượng cao và nhanh chóng, kết hợp công nghệ Dolby.io với THEOplayer.
*   **Tính năng chính:** Phát trực tuyến dựa trên WebRTC với cải tiến âm thanh/video của Dolby, hỗ trợ cuộc gọi/phát sóng nhóm độ trễ thấp, tích hợp với JS frameworks, khử tiếng ồn nâng cao.
*   **Ưu điểm:** Chất lượng âm thanh/video vượt trội, thành phần WebRTC mã nguồn mở miễn phí, độ trễ thấp, tài liệu API/SDK tốt.
*   **Nhược điểm:** Các tính năng nâng cao yêu cầu nền tảng trả phí, thiết lập WebRTC phức tạp cho triển khai lớn, không tối ưu cho HLS/DASH truyền thống.
*   **Cách dùng cơ bản:** `npm install @dolbyio/comms-sdk-web`, sử dụng `VoxeetSDK`.

## So sánh các công cụ phát trực tuyến video

| Công cụ          | Độ trễ      | Chi phí                | Phù hợp nhất cho           | Khả năng mở rộng       |
| :--------------- | :---------- | :--------------------- | :------------------------- | :-------------------- |
| HLS.js           | 3–10s       | Miễn phí               | Trình phát HLS tùy chỉnh    | Cao (phía client)     |
| Video.js         | 3–10s       | Miễn phí               | Phát lại đa định dạng      | Trung bình            |
| Stream API       | <500ms      | Trả phí                | Ứng dụng tương tác         | Rất cao (được quản lý) |
| Daily.co         | <1s         | Freemium               | Hội nghị truyền hình       | Cao (được quản lý)    |
| Node.js/FFmpeg   | Cấu hình được | Miễn phí (hạ tầng)      | Pipeline streaming tự làm | Tự host               |
| Dolby OptiView   | <1s         | Doanh nghiệp (Enterprise)| Phát sóng chất lượng cao   | Cao (được quản lý)    |

## Kết luận
Việc triển khai phát trực tuyến video thời gian thực trong Next.js đòi hỏi phải xem xét nhiều yếu tố như độ trễ, khả năng mở rộng, tài nguyên phát triển và bảo trì dài hạn.

**HLS.js** là một lựa chọn mạnh mẽ cho HTTP Live Streaming, cung cấp các tính năng như ABR, xác thực an toàn và hiệu suất tối ưu.

**Các lựa chọn thay thế** như Video.js, Stream, Daily.co, Node.js/FFmpeg và Dolby OptiView đều có những ưu điểm riêng, phù hợp với các trường hợp sử dụng khác nhau:
*   **HLS.js** hoặc **Video.js** cho phát trực tuyến đáng tin cậy và tương thích rộng rãi.
*   **Daily.co** cho tương tác thời gian thực.
*   **Stream API** để phát triển nhanh chóng với cơ sở hạ tầng được quản lý.
*   **Node.js/FFmpeg** để tùy chỉnh hoàn toàn.
*   **Dolby OptiView** cho trải nghiệm chất lượng cao.

