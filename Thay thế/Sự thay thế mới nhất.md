# 1 Nếu không sử dụng `npm create vrtw`, bạn sẽ cần **nhiều lệnh hơn** và cũng cần thực hiện **nhiều cấu hình thủ công** đáng kể.

`create-vrtw` tự động hóa toàn bộ quá trình, bao gồm:
*   Tạo dự án cơ bản (Vite + React)
*   Cài đặt và cấu hình CSS framework (Tailwind/Bootstrap)
*   Cài đặt các thư viện phụ trợ (React Router, Redux Toolkit, v.v.)
*   Dọn dẹp các file boilerplate không cần thiết

Dưới đây là các bước và số lệnh ước tính nếu bạn làm thủ công để đạt được một dự án tương tự:

---

### 1. Tạo dự án Vite + React cơ bản

*   **Lệnh:** `npm create vite@latest`
    *   _Sau đó, bạn sẽ được hỏi tên dự án, chọn `react` làm framework và `JavaScript` hoặc `TypeScript`._
*   **Lệnh:** `cd <tên-dự-án-của-bạn>`
*   **Lệnh:** `npm install` (hoặc `yarn`, `pnpm`, `bun`)

**Số lệnh ở đây: 3 lệnh** (1 lệnh tạo, 1 lệnh di chuyển thư mục, 1 lệnh cài đặt cơ bản)

---

### 2. Thêm CSS Framework (chọn một trong hai)

**a) Nếu chọn Tailwind CSS v4:**

*   **Lệnh:** `npm install -D tailwindcss postcss autoprefixer`
*   **Lệnh:** `npx tailwindcss init -p`
*   **Lệnh:** `npm install -D @tailwindcss/vite` (để dùng plugin Vite chính thức như `create-vrtw` hỗ trợ)

    *   **Thao tác thủ công:**
        *   Cấu hình file `tailwind.config.js` để thêm các đường dẫn file nguồn (`content`).
        *   Thêm các chỉ thị Tailwind vào file CSS chính của bạn (`index.css` hoặc `App.css`).
        *   Cấu hình `vite.config.js` để sử dụng plugin `@tailwindcss/vite`.

**b) Nếu chọn Bootstrap v5:**

*   **Lệnh:** `npm install bootstrap`

    *   **Thao tác thủ công:**
        *   Import Bootstrap CSS vào file `main.jsx` (hoặc `main.tsx`) của bạn: `import 'bootstrap/dist/css/bootstrap.min.css';`

**Số lệnh ở đây: 1-3 lệnh** tùy framework, **cộng thêm nhiều thao tác thủ công**.

---

### 3. Thêm các thư viện tùy chọn (ví dụ: React Router, Redux Toolkit, v.v.)

Mỗi thư viện sẽ cần một lệnh cài đặt riêng. Bạn có thể gộp chúng thành một lệnh lớn, nhưng về mặt khái niệm, chúng là các bước riêng biệt.

*   **React-Icons:** `npm install react-icons`
*   **React Router:** `npm install react-router-dom`
*   **Redux Toolkit:** `npm install @reduxjs/toolkit react-redux`
*   **Zustand:** `npm install zustand`
*   **Axios:** `npm install axios`

**Số lệnh ở đây: Tùy số lượng thư viện bạn chọn (tối đa 5 lệnh riêng biệt hoặc 1 lệnh gộp)**

---

### 4. Dọn dẹp và cấu hình boilerplate

Đây là phần hoàn toàn thủ công mà `create-vrtw` tự động làm:

*   **Thao tác thủ công:**
    *   Xóa file `src/App.css`.
    *   Xóa các file trong thư mục `public/` (ví dụ: `vite.svg`).
    *   Chỉnh sửa file `src/App.jsx` (hoặc `App.tsx`) để xóa code mặc định và thay bằng một component ứng dụng tối giản, sẵn sàng cho framework CSS bạn đã chọn.
    *   Chỉnh sửa file `src/index.css` để xóa các kiểu dáng mặc định của Vite.
    *   (Nếu dùng TypeScript) Đảm bảo cấu hình TypeScript tương thích với các thư viện mới.

**Số thao tác thủ công: Rất nhiều** (không phải lệnh CLI)

---

### Tổng kết

Nếu không dùng `create-vrtw`, bạn sẽ cần:

*   **Khoảng 4-10 lệnh CLI** (tùy thuộc vào CSS framework và các thư viện phụ trợ).
*   **Rất nhiều thao tác cấu hình và dọn dẹp thủ công** trong các file như `tailwind.config.js`, `vite.config.js`, `main.jsx`, `App.jsx`, `index.css`, thư mục `public`, v.v.

Chính vì những lý do này mà các công cụ scaffolding như `create-vrtw` ra đời để giúp tiết kiệm thời gian và giảm thiểu lỗi cho nhà phát triển.