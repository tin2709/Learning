Dưới đây là bản hướng dẫn hoàn chỉnh và chi tiết nhất về **Convex Full-stack**, kết hợp cả kiến thức về Backend và cách triển khai trên Frontend (React, Next.js, TanStack Query) dựa trên các tài liệu bạn đã cung cấp.

---

# Tổng Quan Hệ Sinh Thái Convex (Backend & Frontend)

Convex là giải pháp Full-stack giúp đồng bộ dữ liệu thời gian thực giữa Backend và Frontend mà không cần viết API (REST/GraphQL) thủ công.

---

## PHẦN 1: CONVEX BACKEND (Cơ sở dữ liệu & Logic)

Mọi logic chạy trên server của Convex đều được viết bằng các **Functions**:

### 1. Ba loại hàm chính
*   **Queries (Truy vấn):** Chỉ đọc dữ liệu. Có tính **Reactive** (tự cập nhật khi DB thay đổi) và được **Cache** tự động.
*   **Mutations (Đột biến):** Ghi dữ liệu (Thêm/Sửa/Xóa). Có tính **Transactional** (đảm bảo hoàn tất mọi thay đổi hoặc không gì cả).
*   **Actions (Hành động):** Dùng để gọi API bên thứ 3 (OpenAI, Stripe...). Chạy được trong môi trường Node.js.

### 2. Thao tác Cơ sở dữ liệu (Database)
*   **Schema:** Định nghĩa cấu trúc dữ liệu tại `convex/schema.ts` để đảm bảo an toàn kiểu dữ liệu (Type-safety).
*   **Indexes:** Luôn dùng Index cho các bảng lớn để tối ưu tốc độ tìm kiếm qua `.withIndex()`.
*   **Search:** Hỗ trợ **Full-text Search** (tìm từ khóa) và **Vector Search** (dùng cho AI).

---

## PHẦN 2: CONVEX REACT CLIENT (Kết nối Frontend)

Để sử dụng Convex trong React, bạn cần cài đặt thư viện: `npm install convex`.

### 1. Thiết lập kết nối
Tạo một `ConvexReactClient` và bao bọc ứng dụng bằng `ConvexProvider`:
```tsx
const convex = new ConvexReactClient(URL_BACKEND);
// Trong Root component:
<ConvexProvider client={convex}>
  <App />
</ConvexProvider>
```

### 2. Các Hooks quan trọng
*   **`useQuery`:** Lấy dữ liệu. Trả về `undefined` khi đang tải và tự động cập nhật UI khi dữ liệu dưới DB thay đổi.
    *   *Mẹo:* Có thể dùng `"skip"` để tạm dừng truy vấn nếu chưa có tham số.
*   **`useMutation`:** Thực hiện thay đổi dữ liệu. Trả về một hàm `async`. Convex tự động thử lại (retry) nếu mạng lỗi.
*   **`useAction`:** Gọi các hàm Action (như gửi email, gọi AI).

### 3. Tính nhất quán (Consistency)
Convex đảm bảo UI luôn hiển thị một trạng thái dữ liệu đồng nhất. Nếu một Mutation thay đổi dữ liệu, tất cả các `useQuery` liên quan trên màn hình sẽ cập nhật cùng một lúc, không bao giờ có tình trạng "xung đột" dữ liệu cũ/mới.

---

## PHẦN 3: TÍCH HỢP FRAMEWORK & THƯ VIỆN

### 1. Next.js (App Router)
*   **Client Components:** Dùng các hooks `useQuery`, `useMutation` như bình thường.
*   **Server Components (SSR):** Convex hỗ trợ render dữ liệu ngay từ server để tối ưu SEO, nhưng cần thiết lập thông qua một `ConvexClientProvider` (Client Component) bao bọc bên ngoài.
*   **Auth:** Tích hợp cực mượt với **Clerk** hoặc **Auth0** thông qua `<ConvexProviderWithClerk>` hoặc các component tương tự.

### 2. TanStack Query (React Query)
Nếu bạn đã quen với TanStack Query, Convex cung cấp thư viện `@convex-dev/react-query` để kết hợp cả hai:
*   **Lợi ích:** Tận dụng được các tính năng của TanStack Query (Loading state, Caching, Error handling) nhưng vẫn có được **dữ liệu thời gian thực** của Convex.
*   **Cơ chế:** Thay vì "Poll" (hỏi liên tục) API, dữ liệu được Convex "Push" (đẩy) về mỗi khi có thay đổi.
*   **Cách dùng:** Sử dụng helper `convexQuery` bên trong hook `useQuery` của TanStack.

---

## PHẦN 4: CÁC KHÁI NIỆM NÂNG CAO

*   **Optimistic Updates:** Cho phép cập nhật UI "tạm thời" ngay khi người dùng nhấn nút (trước khi server phản hồi), giúp ứng dụng cảm giác cực kỳ nhanh.
*   **HTTP Actions:** Tạo các endpoint API để nhận Webhook từ bên ngoài (ví dụ: Stripe thông báo thanh toán thành công).
*   **File Storage:** Convex có sẵn kho lưu trữ file (ảnh, video). Bạn có thể upload trực tiếp từ frontend và quản lý qua hàm Mutation/Action.

---

## TỔNG KẾT QUY TRÌNH PHÁT TRIỂN
1.  **Backend:** Định nghĩa Schema và viết các hàm Query/Mutation trong thư mục `convex/`.
2.  **CLI:** Chạy `npx convex dev` để đồng bộ và sinh ra API object (`api.functions...`).
3.  **Frontend:** Dùng các hooks (`useQuery`, `useMutation`) để gọi các hàm đã viết và hiển thị lên giao diện.
4.  **Auth:** Thêm Clerk/Auth0 nếu cần bảo mật dữ liệu theo từng người dùng.