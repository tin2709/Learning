

## Next.js + MySQL: Hướng dẫn Nhanh (Không cần Express)

Để tương tác với MySQL từ Next.js, bạn có hai cách chính:

1.  **API Routes:** Tạo các endpoint RESTful truyền thống (GET, POST...).
2.  **Server Actions:** Xử lý tác vụ server trực tiếp từ component (cách hiện đại, được khuyến khích).

---

### Bước 1: Cài đặt & Cấu hình

1.  **Chuẩn bị Database:** Đảm bảo bạn có một database MySQL và thông tin kết nối.

2.  **Cài đặt Thư viện:**
    ```bash
    npm install mysql2
    ```

3.  **Tạo file `.env.local`:** Thêm thông tin kết nối database vào file này.
    ```.env.local
    MYSQL_HOST='localhost'
    MYSQL_DATABASE='your_db_name'
    MYSQL_USER='your_username'
    MYSQL_PASSWORD='your_password'
    ```

---

### Bước 2: Quản lý Kết nối (Connection Pool)

Tạo file `lib/db.ts` để quản lý và tái sử dụng kết nối, giúp tối ưu hiệu suất.

```typescript
// lib/db.ts
import mysql from 'mysql2/promise';

let pool: mysql.Pool;

export function getDbPool() {
  if (!pool) {
    pool = mysql.createPool({
      host: process.env.MYSQL_HOST,
      user: process.env.MYSQL_USER,
      password: process.env.MYSQL_PASSWORD,
      database: process.env.MYSQL_DATABASE,
      waitForConnections: true,
      connectionLimit: 10,
    });
  }
  return pool;
}
```

---

### Cách 1: Tạo API Route (RESTful)

Tạo file `app/api/products/route.ts` để định nghĩa các endpoint.

```typescript
// app/api/products/route.ts
import { NextResponse } from 'next/server';
import { getDbPool } from '@/lib/db';

// Lấy danh sách sản phẩm
export async function GET() {
  const pool = getDbPool();
  const [rows] = await pool.query('SELECT * FROM products');
  return NextResponse.json(rows);
}

// Tạo sản phẩm mới
export async function POST(request: Request) {
  const { name, price } = await request.json();
  const pool = getDbPool();
  
  // Dùng `?` để chống SQL Injection
  const query = 'INSERT INTO products (name, price) VALUES (?, ?)';
  await pool.execute(query, [name, price]);
  
  return NextResponse.json({ message: 'Product created' }, { status: 201 });
}
```

---

### Cách 2: Dùng Server Actions (Hiện đại)

Đây là cách xử lý form submit hiệu quả nhất.

#### 1. Tạo file Action

Tạo file `app/actions/product-actions.ts`.
```typescript
// app/actions/product-actions.ts
'use server';

import { getDbPool } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function createProduct(formData: FormData) {
  const name = formData.get('name') as string;
  const price = Number(formData.get('price'));

  if (!name || isNaN(price)) {
    return { message: 'Dữ liệu không hợp lệ.', success: false };
  }

  try {
    const pool = getDbPool();
    const query = 'INSERT INTO products (name, price) VALUES (?, ?)';
    await pool.execute(query, [name, price]);

    revalidatePath('/products'); // Cập nhật lại trang sản phẩm
    return { message: 'Thêm sản phẩm thành công!', success: true };
  } catch (e) {
    return { message: 'Lỗi server.', success: false };
  }
}
```

#### 2. Gọi Action từ Component

Tạo file `app/products/ProductForm.tsx`.
```tsx
// app/products/ProductForm.tsx
'use client';

import { useActionState, useRef } from 'react';
import { createProduct } from '@/app/actions/product-actions';

export function ProductForm() {
  const [state, formAction, isPending] = useActionState(createProduct, null);
  const formRef = useRef<HTMLFormElement>(null);

  if (state?.success) {
    formRef.current?.reset();
  }

  return (
    <form ref={formRef} action={formAction} className="space-y-4">
      <input name="name" placeholder="Tên sản phẩm" className="border p-2" required />
      <input name="price" type="number" placeholder="Giá" className="border p-2" required />
      <button type="submit" disabled={isPending} className="bg-blue-500 text-white p-2">
        {isPending ? 'Đang thêm...' : 'Thêm'}
      </button>
      {state?.message && <p>{state.message}</p>}
    </form>
  );
}
```

---

### Tổng kết

Với Next.js, bạn không cần một server Express riêng. Logic backend được tích hợp ngay trong thư mục `app` thông qua **API Routes** (cho API công khai) và **Server Actions** (cho các tương tác trong app), giúp kiến trúc đơn giản và hiệu quả.