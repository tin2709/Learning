Dưới đây là nội dung bài viết đã được chuyển đổi sang định dạng Markdown (`.md`), tối ưu cho các nền tảng như GitHub, Dev.to hoặc Blog cá nhân.

---

# Frontend Security: Khi Code Của Bạn Trở Thành Cửa Hậu Cho Hacker

> *"Frontend đang xử lý authentication, data access và business logic — nhưng ai đang nghĩ về security?"*

Tôi vẫn nhớ cái thời mà một frontend developer chỉ cần lo CSS đẹp, JavaScript chạy mượt là xong. Security? Đó là việc của backend team, của DevOps, của "người khác".

Điển hình năm 2025 đã dập tắt hoàn toàn quan niệm đó. Chỉ trong quý đầu năm, chúng ta đã chứng kiến một loạt lỗ hổng bảo mật nghiêm trọng tấn công trực tiếp vào lớp frontend — nơi mà hầu hết các team vẫn đang để ngỏ. 

Bài viết này không phải để dọa bạn. Mà để thay đổi tư duy.

---

## 1. Bức tranh toàn cảnh: Frontend không còn chỉ là "giao diện"

Hãy nhìn lại kiến trúc của một ứng dụng web hiện đại. React, Vue, Angular, Next.js — những framework này đang đảm nhận:
- **Xử lý authentication flows**: login, logout, refresh token, session management.
- **Kiểm soát data access**: quyết định user nào thấy được dữ liệu gì.
- **Chứa business logic**: validation, calculation, workflow routing.
- **Gọi API trực tiếp**: với credentials được lưu trữ ngay trên client.
- **Quản lý state nhạy cảm**: thông tin thanh toán, dữ liệu cá nhân, token phân quyền.

**Nói ngắn gọn:** Browser đã trở thành một phần của attack surface. Và hầu hết các team đang bảo vệ nó bằng... không có gì đáng kể.

---

## 2. Case Study #1: CVE-2025-29927 — Khi Một Header Phá Vỡ Tất Cả

### Chuyện gì đã xảy ra?
Ngày 21/3/2025, nhóm bảo mật của Next.js công bố lỗ hổng **CVE-2025-29927** — lỗi bypass authorization trong middleware.

Vấn đề nằm ở một internal HTTP header có tên `x-middleware-subrequest`. Header này vốn để ngăn chặn infinite loop trong middleware, nhưng framework lại tin tưởng nó ngay cả khi nó đến từ người dùng bên ngoài.

**Cách thức tấn công:**
Attacker chỉ cần gửi request với header:
```http
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```
Và toàn bộ middleware — bao gồm authentication check, session validation — bị bỏ qua hoàn toàn.

### Phạm vi ảnh hưởng
- **Next.js 15.x**: 15.0.0 đến 15.2.2
- **Next.js 14.x**: 14.0.0 đến 14.2.24
- **Next.js 11.x đến 13.x**: 11.1.4 đến 13.5.6

### Cách khắc phục
1. **Upgrade ngay:** Lên phiên bản 15.2.3, 14.2.25 hoặc các bản patch tương ứng.
2. **Cấu hình Proxy (Nginx/Apache):** Strip header nguy hiểm trước khi đến app server.

```nginx
# Nginx fix
proxy_set_header x-middleware-subrequest "";
```

---

## 3. Case Study #2: CVE-2025-55182 — React2Shell và Supply Chain Attack

Nếu CVE-2025-29927 là cú đấm thẳng, thì **CVE-2025-55182** là nhát dao đâm từ phía sau. 

Một dự án React trung bình có hàng trăm dependencies. Khi một package bị compromise, toàn bộ ứng dụng của bạn sẽ bị chiếm quyền kiểm soát. Frontend là mục tiêu béo bở vì:
- Dependencies thường không được audit kỹ.
- `node_modules` được coi là "black box".
- Security scanning thường chỉ tập trung vào backend.

---

## 4. Checklist Security Thực Tế Cho Frontend Developer

### ✅ 4.1 Token Storage — Đừng để tiền trong túi quần thủng
```javascript
// ❌ SAI — XSS có thể đọc được ngay lập tức
localStorage.setItem('access_token', token);

// ✅ ĐÚNG — Dùng HttpOnly cookie (xử lý phía Server)
// Cookie này không thể truy cập bằng JavaScript, chặn đứng nguy cơ đánh cắp token qua XSS.
```

### ✅ 4.2 Content Security Policy (CSP) — Không phải optional
CSP là lớp bảo vệ quan trọng nhất chống XSS. Hãy cấu hình nó trong header:
```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}'; connect-src 'self' https://api.yourdomain.com;
```

### ✅ 4.3 Defense in Depth — Đừng tin tưởng một lớp duy nhất
Đừng bao giờ coi Middleware là lớp bảo mật cuối cùng. 

```typescript
// 1. Lớp Middleware (Tiện lợi/UX)
export function middleware(request: NextRequest) {
  // Check nhanh ở đây
}

// 2. Lớp Server Component/API Route (Bắt buộc)
async function AdminPage() {
  const session = await getServerSession(); // Validate lại ở đây
  if (!session?.user?.isAdmin) redirect('/403');
}
```

### ✅ 4.4 Dependency Audit — Làm thường xuyên
Thêm bước này vào CI/CD pipeline của bạn:
```bash
# Chạy trong PR
npm audit --audit-level=moderate

# Hoặc dùng công cụ chuyên dụng
npx snyk test
```

### ✅ 4.5 Subresource Integrity (SRI)
Khi load script từ CDN, hãy luôn dùng mã hash để đảm bảo file không bị hacker sửa đổi trên server của bên thứ ba.
```html
<script src="https://cdn.example.com/library.js"
        integrity="sha384-abc123..."
        crossorigin="anonymous"></script>
```

---

## 5. Câu hỏi mà mọi team nên tự hỏi

1. Nếu middleware bị bypass, điều tệ nhất có thể xảy ra là gì?
2. Nếu một npm dependency bị compromise, nó có thể làm gì trong trình duyệt của user?
3. Tokens và credentials của chúng ta có đang nằm ở `localStorage` không?
4. Lần cuối cùng chúng ta audit các script bên thứ ba (analytics, chatbot) là khi nào?

---

## Kết luận: Security Là Trách Nhiệm Của Cả Team

Trong năm 2025, không thể tiếp tục coi security là "việc của người khác". **Defense in depth** (Phòng thủ đa tầng) là chìa khóa: khi một lớp thất bại (như lỗi Next.js middleware), các lớp khác (Server-side validation, CSP, HttpOnly cookies) vẫn phải đủ mạnh để bảo vệ dữ liệu.

Tôi không yêu cầu bạn trở thành chuyên gia bảo mật, nhưng hãy chọn **không build những hệ thống thiếu an toàn**.

---
*Người viết: [Tên của bạn/Brand của bạn]*
*Ngày cập nhật: 24/05/2025*