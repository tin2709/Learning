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