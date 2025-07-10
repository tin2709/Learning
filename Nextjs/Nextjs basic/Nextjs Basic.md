# I  Giới thiệu về Next.js

## 1. Next.js là gì?

- Next.js là fullstack framework cho React.js được tạo ra bởi Vercel (trước đây là ZEIT).
- Next có thể làm server như Express.js bên Node.js và có thể làm client như React.js

## 2. Next.js giải quyết vấn đề gì?

### Đầu tiên là render website ở Server nên thân thiện với SEO

React.js thuần chỉ là client side rendering, nhanh thì cũng có nhanh nhưng không tốt cho SEO. Ai nói với bạn rằng sài React.js thuần vẫn lên được top google ở nhiều thì đó là lừa đảo (hoặc họ chỉ đang nói 1 nữa sự thật)

Next.js hỗ trợ server side rendering, nghĩa là khi người dùng request lên server thì server sẽ render ra html rồi trả về cho người dùng. Điều này giúp cho SEO tốt hơn.

### Tích hợp nhiều tool mà React.js thuần không có

- Tối ưu image, font, script
- CSS module
- Routing
- Middleware
- Server Action
- SEO ...

### Thống nhất về cách viết code

Ở React.js, có quá nhiều cách viết code và không có quy chuẩn.

Ví dụ:

- Routing có thể dùng React Router Dom hoặc TanStack Router.
- Nhiều cách bố trí thư mục khác nhau

Dẫn đến sự không đồng đều khi làm việc nhóm và khó bảo trì.

Next.js giúp bạn thống nhất về cách viết code theo chuẩn của họ => giải quyết phần nào đó các vấn đề trên

### Đem tiền về cho Vercel 🙃

Ngày xưa các website thường đi theo hướng Server Side Rendering kiểu Multi Page Application (MPA) như PHP, Ruby on Rails, Django, Express.js ... Ưu điểm là web load nhanh và SEO tốt, nhưng nhược điểm là UX hay bị chớp chớp khi chuyển trang và khó làm các logic phức tạp bên client.

Sau đó React.js, Angular, Vue ra đời, đi theo hướng Single Page Application (SPA) giải quyết được nhược điểm của MPA, nhưng lại tạo ra nhược điểm mới là SEO kém và load chậm ở lần đầu.

Vercel là công ty cung cấp các dịch vụ phía Server như hosting website, serverless function, database, ...và họ cũng là công ty đầu tiên khởi xướng trào lưu "quay trở về Server Side Rendering" .

Vì thế họ tạo ra Next.js, vừa để khắc phục nhược điểm của SPA truyền thống, vừa gián tiếp bán các sản phẩm dịch vụ của họ. Ví dụ Next.js chạy trên dịch vụ Edge Runtime của họ sẽ có độ trễ thấp hơn so với chạy trên Node.js



# 2 Cách tạo Sitemap trong Next.js bằng cách sử dụng gói `next-sitemap`

README này sẽ hướng dẫn bạn cách tự động hóa quy trình tạo sitemap cho ứng dụng Next.js của mình bằng cách sử dụng gói npm `next-sitemap`. Việc có một sitemap được cập nhật thường xuyên là rất quan trọng để cải thiện SEO (Tối ưu hóa Công cụ Tìm kiếm) cho website của bạn.

## Giới thiệu về gói NPM `next-sitemap`

`next-sitemap` là một gói phổ biến và dễ sử dụng được thiết kế đặc biệt để tạo sitemap (`sitemap.xml`) và file `robots.txt` cho các dự án Next.js. Nó được Vishnu Sankar tạo ra và bảo trì.

*   **Kho lưu trữ GitHub:** [https://github.com/iamvishnusankar/next-sitemap](https://github.com/iamvishnusankar/next-sitemap) (Với hơn 3.4k sao)
*   **Mục đích:** Mục đích của gói này rất đơn giản nhưng cực kỳ quan trọng đối với SEO. Nó giúp các công cụ tìm kiếm dễ dàng khám phá và lập chỉ mục tất cả các trang trên website của bạn bằng cách cung cấp một danh sách các URL.

## Bắt đầu

Hãy cùng đi qua các bước cơ bản để thiết lập `next-sitemap`.

### Bước 1: Cài đặt gói

Mở terminal trong thư mục gốc của dự án Next.js của bạn và chạy lệnh sau:

```bash
npm install next-sitemap
# Hoặc dùng yarn: yarn add next-sitemap
# Hoặc dùng pnpm: pnpm add next-sitemap
```

### Bước 2: Tạo tệp cấu hình `next-sitemap.config.js`

Tạo một tệp có tên chính xác là `next-sitemap.config.js` trong thư mục gốc của dự án của bạn (ngang hàng với `package.json`, `next.config.js`, v.v.).

Nội dung cơ bản của tệp này sẽ như sau:

```javascript
/** @type {import('next-sitemap').IConfig} */
module.exports = {
  siteUrl: process.env.SITE_URL || 'https://example.com', // Đặt URL website của bạn
  generateRobotsTxt: true, // (Tùy chọn) Có tạo file robots.txt không
  // ...các tùy chọn khác
}
```

**Lưu ý:** Đảm bảo bạn thay thế `'https://example.com'` bằng URL thực tế của website của bạn. Bạn nên sử dụng biến môi trường (`process.env.SITE_URL`) để linh hoạt hơn giữa các môi trường (development, staging, production).

### Bước 3: Thêm Script vào `package.json`

Thêm hoặc sửa đổi phần `scripts` trong tệp `package.json` của bạn để chạy `next-sitemap` sau khi quá trình build của Next.js hoàn tất.

```json
"scripts": {
  "build": "next build",
  "postbuild": "next-sitemap" // Thêm dòng này
}
```

**Giải thích:** Script `postbuild` là một script đặc biệt trong `package.json` sẽ tự động chạy *sau* khi script `build` (hoặc bất kỳ script nào có tiền tố `pre` hoặc `post` tương ứng) hoàn thành thành công.

**Cảnh báo khi dùng `pnpm`:**
Khi sử dụng `pnpm`, script `postbuild` có thể không hoạt động như mong đợi do cách `pnpm` quản lý các hook. Nếu gặp vấn đề, bạn có thể thay thế script `build` thành:

```json
"scripts": {
  "build": "next build && next-sitemap" // Thay thế dòng build bằng dòng này
}
```

### Bước 4: Chạy lệnh Build

Chạy lệnh build Next.js như bình thường:

```bash
npm run build
# Hoặc: yarn build
# Hoặc: pnpm build
```

Quá trình này sẽ chạy `next build`, và sau khi nó hoàn thành, script `postbuild` (hoặc phần `&& next-sitemap` nếu bạn đã sửa đổi script `build` cho pnpm) sẽ được thực thi.

### Bước 5: Kiểm tra kết quả

Sau khi lệnh build chạy thành công, bạn sẽ thấy các tệp `sitemap.xml` và `robots.txt` được tạo ra trong thư mục `public` của dự án:

```
| your-nextjs-project/
    | app/ (hoặc pages/)
    | public/
        | sitemap.xml   <-- Tệp được tạo
        | robots.txt    <-- Tệp được tạo (nếu generateRobotsTxt: true)
    | next-sitemap.config.js
    | package.json
    | ...other files
```

Bạn có thể mở các tệp này để xem nội dung được tạo. Nếu bạn không muốn tạo tệp `robots.txt`, hãy đặt `generateRobotsTxt: false` trong tệp cấu hình.

## Cấu hình Nâng cao cho Sitemap

`next-sitemap` cung cấp các tùy chọn cấu hình mạnh mẽ để tùy chỉnh sitemap của bạn.

### Tùy chỉnh `priority` và `changefreq`

Theo mặc định, gói sẽ gán cùng mức độ ưu tiên (`priority`) và tần suất thay đổi (`changefreq`) cho tất cả các trang. Bạn có thể thay đổi điều này bằng cách sử dụng hàm `transform` trong tệp cấu hình.

**Ví dụ:** Đặt `priority` của trang chủ (`/`) là 1.0 (cao nhất) và các trang khác là 0.8, cùng với `changefreq` riêng cho trang chủ.

```javascript
/** @type {import('next-sitemap').IConfig} */
module.exports = {
  siteUrl: process.env.SITE_URL || 'https://example.com',
  changefreq: 'daily', // Mặc định cho các trang
  priority: 0.8,     // Mặc định cho các trang
  sitemapSize: 5000, // Tùy chọn: Chia sitemap thành nhiều file nếu quá lớn
  generateRobotsTxt: true,
  transform: async (config, path) => {
    let priority = config.priority;
    let changefreq = config.changefreq;

    // Đặt ưu tiên cao hơn cho trang chủ
    if (path === '/') {
      priority = 1.0; // Ưu tiên cao nhất
      changefreq = 'hourly'; // Ví dụ: Trang chủ thay đổi thường xuyên hơn
    }

    // Bạn có thể thêm logic tùy chỉnh khác ở đây dựa trên 'path'
    // Ví dụ: if (path.startsWith('/blog/')) { priority = 0.9; }

    return {
      loc: path, // => đây sẽ được xuất thành http(s)://<config.siteUrl>/<path>
      changefreq: changefreq, // Sử dụng giá trị đã tùy chỉnh
      priority: priority,     // Sử dụng giá trị đã tùy chỉnh
      lastmod: config.autoLastmod ? new Date().toISOString() : undefined, // Tùy chọn: Ngày sửa đổi cuối cùng
      alternateRefs: config.alternateRefs ?? [], // Tùy chọn: Cho các phiên bản ngôn ngữ khác
    };
  },
  // ...các tùy chọn khác
}
```

Hàm `transform` nhận cấu hình hiện tại (`config`) và đường dẫn của trang (`path`) làm đối số, cho phép bạn trả về cấu trúc dữ liệu tùy chỉnh cho mục nhập sitemap của trang đó.

### Loại trừ các Trang khỏi Sitemap

Nếu có các trang bạn không muốn đưa vào sitemap (ví dụ: trang admin, trang test, các trang động không cần thiết cho SEO), bạn có thể sử dụng tùy chọn `exclude`.

**Ví dụ:** Loại trừ tất cả các trang có đường dẫn bắt đầu bằng `/blank/`.

```javascript
/** @type {import('next-sitemap').IConfig} */
module.exports = {
  siteUrl: process.env.SITE_URL || 'https://example.com',
  changefreq: 'daily',
  priority: 0.8,
  sitemapSize: 5000,
  generateRobotsTxt: true,
  transform: async (config, path) => {
    // ... giữ nguyên logic transform như trên nếu bạn muốn tùy chỉnh priority/changefreq
     let priority = config.priority;
     let changefreq = config.changefreq;
     if (path === '/') {
       priority = 1.0;
       changefreq = 'hourly';
     }
    return {
       loc: path,
       changefreq: changefreq,
       priority: priority,
       lastmod: config.autoLastmod ? new Date().toISOString() : undefined,
       alternateRefs: config.alternateRefs ?? [],
     };
  },
  // Thêm mảng các đường dẫn cần loại trừ
  exclude: ['/blank/*'], // Sử dụng glob pattern để loại trừ tất cả các trang trong thư mục /blank/
  // ...các tùy chọn khác
}
```

Tùy chọn `exclude` nhận một mảng các chuỗi hoặc glob pattern để chỉ định các đường dẫn cần bỏ qua khi tạo sitemap.

## Kết luận

Sử dụng gói `next-sitemap` là một cách hiệu quả và đơn giản để tự động hóa việc tạo `sitemap.xml` và `robots.txt` cho ứng dụng Next.js của bạn. Bằng cách làm theo các bước cơ bản và tận dụng các tùy chọn cấu hình nâng cao như `transform` và `exclude`, bạn có thể đảm bảo sitemap của mình luôn chính xác, được cập nhật và tối ưu cho các công cụ tìm kiếm, góp phần cải thiện SEO tổng thể cho website của bạn.

Chúc bạn thành công!

---

*(README này được tạo dựa trên nội dung được cung cấp về gói `next-sitemap`.)*

Chắc chắn rồi! Dưới đây là giải thích chi tiết về Turbopack một cách dễ hiểu.

# 2 Turbopack

**Turbopack là một bundler (trình đóng gói) mã nguồn mở, hiệu năng cực cao, được viết bằng ngôn ngữ Rust.** Nó được tạo ra bởi Vercel (công ty đứng sau Next.js) với mục tiêu trở thành người kế nhiệm tinh thần của Webpack, mang lại tốc độ nhanh hơn đáng kể cho quá trình phát triển và build ứng dụng web.

---

### Giải thích chi tiết

#### 1. Turbopack là gì?

Hãy tưởng tượng khi bạn xây dựng một trang web hiện đại, bạn không chỉ viết HTML, CSS, JavaScript đơn giản. Bạn dùng TypeScript, JSX (trong React), SASS, CSS Modules, và rất nhiều thư viện khác. Trình duyệt không thể hiểu trực tiếp tất cả những thứ này.

**Bundler** (như Webpack, Vite, hay Turbopack) là công cụ làm nhiệm vụ "dọn dẹp và đóng gói":

*   Nó lấy tất cả các file mã nguồn của bạn (TypeScript, JSX, CSS, hình ảnh,...).
*   Biên dịch chúng thành định dạng mà trình duyệt có thể hiểu được (chủ yếu là JavaScript, CSS, HTML).
*   Gộp chúng lại thành các file tối ưu để tải nhanh hơn trên trình duyệt.

**Turbopack chính là một bundler thế hệ mới, được thiết kế với ưu tiên hàng đầu là tốc độ.**

#### 2. Tại sao Turbopack ra đời?

*   **Webpack đã trở nên chậm chạp:** Webpack là một công cụ cực kỳ mạnh mẽ và phổ biến, nhưng khi các dự án ngày càng lớn, thời gian khởi động server dev và thời gian build có thể kéo dài từ vài giây đến vài phút. Điều này làm chậm chu trình làm việc của lập trình viên.
*   **Sức mạnh của Rust:** Turbopack được viết bằng Rust, một ngôn ngữ lập trình hệ thống nổi tiếng về hiệu năng, an toàn bộ nhớ và khả năng xử lý song song. Điều này cho phép Turbopack tận dụng tối đa phần cứng để thực thi các tác vụ nhanh hơn nhiều so với các công cụ viết bằng JavaScript (như Webpack).
*   **Kiến trúc mới:** Turbopack được xây dựng trên một kiến trúc gọi là **biên dịch tăng trưởng (incremental compilation)**.

#### 3. Các đặc điểm nổi bật của Turbopack

1.  **Tốc độ vượt trội (The main selling point):**
    *   Vercel tuyên bố Turbopack **nhanh hơn Webpack 700 lần** và **nhanh hơn Vite 10 lần** trong các tác vụ cập nhật lớn (cập nhật khi code thay đổi).
    *   Thời gian khởi động server dev gần như là tức thì, ngay cả với các dự án lớn.

2.  **Biên dịch tăng trưởng (Incremental by design):**
    *   Đây là "bí mật" đằng sau tốc độ của Turbopack. Thay vì build lại toàn bộ ứng dụng mỗi khi bạn thay đổi một file, Turbopack chỉ build lại những gì thực sự thay đổi.
    *   Nó thực hiện việc này ở cấp độ hàm (function-level caching). Điều này có nghĩa là nó sẽ không bao giờ thực hiện cùng một công việc hai lần.

3.  **Hỗ trợ sẵn các công nghệ hiện đại:**
    *   Không cần cấu hình phức tạp, Turbopack hỗ trợ sẵn TypeScript, JSX, CSS, CSS Modules, WebAssembly, v.v.

4.  **Tối ưu hóa cho Development:**
    *   Tốc độ cập nhật cực nhanh khi bạn lưu file (Hot Module Replacement - HMR) giúp trải nghiệm lập trình mượt mà hơn rất nhiều.

#### 4. So sánh nhanh: Turbopack vs. Webpack vs. Vite

| Tiêu chí | Turbopack | Vite | Webpack |
| :--- | :--- | :--- | :--- |
| **Ngôn ngữ chính** | **Rust** | Go (cho esbuild) & JS | JavaScript |
| **Kiến trúc Dev** | **Rust-based bundler** | Native ES Modules (ESM) | Bundling toàn bộ |
| **Hiệu năng** | **Cực cao**, nhất là với các dự án lớn | Rất cao, khởi động nhanh | Chậm hơn, đặc biệt khi dự án lớn |
| **Tình trạng** | **Beta** | Ổn định (Stable) | Rất ổn định, hệ sinh thái lớn |
| **Người tạo** | Vercel (bởi tác giả của Webpack) | Evan You (tác giả của Vue.js) | Tobias Koppers & cộng đồng |

*   **Turbopack vs. Webpack:** Turbopack được coi là người kế nhiệm. Nó được tạo ra bởi chính người đã tạo ra Webpack (Tobias Koppers) để giải quyết các vấn đề về hiệu năng của Webpack.
*   **Turbopack vs. Vite:** Đây là đối thủ cạnh tranh trực tiếp. Cả hai đều cực kỳ nhanh. Vite nhanh nhờ sử dụng Native ES Modules của trình duyệt trong môi trường dev. Turbopack nhanh nhờ kiến trúc biên dịch tăng trưởng viết bằng Rust.

#### 5. Hiện trạng và cách sử dụng

*   **Tình trạng:** Hiện tại, Turbopack vẫn đang trong giai đoạn **Beta**. Nó chưa hoàn toàn ổn định để thay thế hoàn toàn Webpack trong môi trường production cho mọi trường hợp.
*   **Cách sử dụng:** Cách phổ biến nhất để trải nghiệm Turbopack là thông qua **Next.js**. Bạn có thể bật nó cho server development bằng lệnh:
    ```bash
    next dev --turbo
    ```
    Bạn sẽ thấy thời gian khởi động và cập nhật được cải thiện một cách rõ rệt.

### Kết luận

**Turbopack là một bước tiến lớn trong thế giới công cụ web, hứa hẹn một tương lai nơi việc build và phát triển các ứng dụng web phức tạp sẽ nhanh như chớp.** Mặc dù vẫn còn trong giai đoạn Beta, nó đã cho thấy tiềm năng to lớn và đang được tích cực phát triển để trở thành bundler mặc định cho Next.js và có thể là cho cả hệ sinh thái web rộng lớn hơn.
