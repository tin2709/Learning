
# 1 Streaming Phía Máy Chủ Tuyệt Vời Trên Các Framework
Sponsor by https://www.telerik.com/blogs/fantastic-framework-server-streaming?ref=dailydev

**Bởi Jonathan Gamble**
Ngày 16 tháng 5, 2025

---

Khi bạn có dữ liệu tải chậm, không thiết yếu trên trang web của mình, việc truyền tải dữ liệu (streaming) có thể là một cách tuyệt vời để giữ cho trang nhanh hơn và tương tác tốt hơn.

**TL;DR**

Chúng tôi xây dựng và phân tích một ví dụ về streaming trong tất cả các framework lớn có hỗ trợ tính năng này.

## HTTP Streaming Là Gì?

Khi có một số dữ liệu trong trang web mất nhiều thời gian hơn để tải so với dữ liệu thiết yếu, điều này có thể làm chậm trang. Streaming cho phép chúng ta tải dữ liệu quan trọng trước, sau đó mới tải dữ liệu không thiết yếu sau.

Streaming đã chứng minh khả năng giúp cải thiện SEO, bao gồm Thời gian hiển thị nội dung đầu tiên (First Contentful Paint) nhanh hơn, Thời gian hiển thị nội dung lớn nhất (Largest Contentful Paint) tốt hơn và Thời gian đến byte đầu tiên (Time to First Byte).

Về cơ bản, streaming tải dữ liệu chậm sau cùng, và sau đó JavaScript sẽ chèn dữ liệu vào đúng vị trí khi toàn bộ trang đã được tải. Đó là lời giải thích đơn giản nhất.

## Ví dụ: Trang Blog

Hãy tưởng tượng bạn đang xem một bài đăng blog có nhiều bình luận. Bạn cần xem dữ liệu bài đăng blog trước, để các mục như tiêu đề và mô tả có thể hiển thị ngay lập tức trong `<head>` và phần header của bài viết.

Nếu bạn có nhiều bình luận, sẽ tốt hơn nếu tách truy vấn cơ sở dữ liệu để lấy bình luận sau. Bình luận ít quan trọng hơn và có thể hiển thị sau khi dữ liệu thiết yếu đã được tải xong.

## Ví dụ ứng dụng Todo

Trong các ứng dụng của chúng ta, chúng ta sẽ lấy một mục todo ngẫu nhiên từ một API ví dụ bên ngoài (bằng tiếng Latin).

```typescript
type Todo = {
    title: string
};

export const getTodo = async () => {
    const randomTodo = Math.floor(Math.random() * 200) + 1;
    return await fetch(`https://jsonplaceholder.typicode.com/todos/${randomTodo}`)
        .then(r => r.json()) as Todo;
};
```

API này có 200 mục todo khác nhau, vì vậy chúng ta sẽ chọn ngẫu nhiên một mục mỗi khi trang được tải. Nếu bạn làm mới trang, bạn sẽ nhận được một mục mới.

## Triển khai theo Framework

Dưới đây là cách triển khai streaming trong các framework phổ biến:

### React TSX (Next.js)

Mặc dù cá nhân tôi không thích React và JSX, tôi rất khen ngợi nhóm Next.js tại Vercel. Next.js gần đây gây nhiều tranh cãi, nhưng streaming chắc chắn là một điểm mạnh.

**Todo Component**

```tsx
// todo.tsx

'use server';

type Todo = {
    title: string
};

export const getTodo = async () => {
    const randomTodo = Math.floor(Math.random() * 200) + 1;
    return await fetch(`https://jsonplaceholder.typicode.com/todos/${randomTodo}`, {
        cache: 'no-cache' // Cần tùy chọn này để tránh cache
    })
        .then(r => r.json()) as Todo;
};

export default async function Todo() {
    const todo = await getTodo();
    return (
        <h2>{todo.title}</h2>
    );
}
```

Chúng ta đang sử dụng một server component với `'use server'`. Lưu ý chúng ta phải thêm tùy chọn `no-cache` để ngăn Next.js cache trang của chúng ta.

**Loading Spinner**

Tôi cũng sử dụng một loading spinner (biểu tượng đang tải). Hãy nhớ rằng, đây là vẻ đẹp của streaming. Chúng ta tải trực tiếp từ máy chủ và sau đó hiển thị trong trình duyệt.

```tsx
// loading.tsx

export default function Loading() {
    return (
        <div role="status">
            <svg
                aria-hidden="true"
                className="w-8 h-8 text-gray-200 animate-spin dark:text-gray-600 fill-blue-600"
                viewBox="0 0 100 101"
                fill="none"
                xmlns="http://www.w3.org/2000/svg"
            >
                <path
                    d="M100 50.5908C100 78.2051 77.6142 100.591 50 100.591C22.3858 100.591 0 78.2051 0 50.5908C0 22.9766 22.3858 0.59082 50 0.59082C77.6142 0.59082 100 22.9766 100 50.5908ZM9.08144 50.5908C9.08144 73.1895 27.4013 91.5094 50 91.5094C72.5987 91.5094 90.9186 73.1895 90.9186 50.5908C90.9186 27.9921 72.5987 9.67226 50 9.67226C27.4013 9.67226 9.08144 27.9921 9.08144 50.5908Z"
                    fill="currentColor"
                />
                <path
                    d="M93.9676 39.0409C96.393 38.4038 97.8624 35.9116 97.0079 33.5539C95.2932 28.8227 92.871 24.3692 89.8167 20.348C85.8452 15.1192 80.8826 10.7238 75.2124 7.41289C69.5422 4.10194 63.2754 1.94025 56.7698 1.05124C51.7666 0.367541 46.6976 0.446843 41.7345 1.27873C39.2613 1.69328 37.813 4.19778 38.4501 6.62326C39.0873 9.04874 41.5694 10.4717 44.0505 10.1071C47.8511 9.54855 51.7191 9.52689 55.5402 10.0491C60.8642 10.7766 65.9928 12.5457 70.6331 15.2552C75.2735 17.9648 79.3347 21.5619 82.5849 25.841C84.9175 28.9121 86.7997 32.2913 88.1811 35.8758C89.083 38.2158 91.5421 39.6781 93.9676 39.0409Z"
                    fill="currentFill"
                />
            </svg>
            <span className="sr-only">Đang tải...</span>
        </div>
    );
}
```

*Lưu ý*: Spinner ví dụ này được lấy từ Flowbite và yêu cầu Tailwind CSS. Bạn cũng có thể thử component Loader có sẵn trong KendoReact Free.

**Suspense**

React đã có một element có thể tải dữ liệu chậm. Nó được gọi là `<Suspense />`.

```tsx
import { Suspense } from "react";
import Todo from "./todo";
import Loading from "./loading";

export default function Home() {
  return (
    <main className="flex flex-col justify-center items-center mt-5 gap-3">
      <h1 className="text-2xl">Todo</h1>
      <Suspense fallback={<Loading />}>
       <Todo />       
      </Suspense>
    </main>
  );
}
```

Bằng cách bọc element của chúng ta trong Suspense, và sử dụng loader của chúng ta làm fallback, mọi thứ hoạt động hoàn hảo.

*   [Demo: Vercel](https://nextjs-streaming-todo-app.vercel.app/)
*   [Kho mã nguồn: GitHub](https://github.com/jonathangamble/nextjs-streaming-todo-app)
*   [Tài liệu: Loading UI and Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)

### SvelteKit

SvelteKit không có server components, nhưng có page loaders.

**Page Server Loader**

```typescript
import type { PageServerLoad } from "./$types";

type Todo = {
    title: string
};

const getTodo = async () => {
    const randomTodo = Math.floor(Math.random() * 200) + 1;
    return await fetch(`https://jsonplaceholder.typicode.com/todos/${randomTodo}`)
        .then(r => r.json()) as Todo;
};

export const load: PageServerLoad = () => {
    return {
        todos: getTodo() // Trả về Promise trực tiếp để streaming
    };
};
```

Khi chúng ta trả về hàm async trực tiếp, hàm sẽ được streaming đến trình duyệt khi nó được giải quyết (resolved).

```typescript
export const load: PageServerLoad = () => {
		return {
				post: getPost()
		};
};
```

Trừ khi chúng ta có dữ liệu thiết yếu cần tải trước.

```typescript
// Nếu cần chờ dữ liệu thiết yếu
export const load: PageServerLoad = async () => { // Thêm async vào đây
		const post = await getPost();
		return {
				post
		};
}

// HOẶC

export const load: PageServerLoad = () => {
		return {
				post: await getPost() // Chờ Promise giải quyết
		};
}
```

*Lưu ý*: Đảm bảo Promise không thể bị từ chối (rejected) trước khi trả về hàm để xử lý lỗi đúng cách.

**Page Component**

Chúng ta có thể lấy dữ liệu trang bằng cách sử dụng `await` bên trong component của mình. Chúng ta bắt lỗi nếu cần hoặc hiển thị loader. Thật tuyệt vời!

```svelte
<script lang="ts">
  import type { PageData } from "./$types"; // Sử dụng PageData thay vì PageProps
  import Loading from "./loading.svelte";

  let { data }: { data: PageData } = $props(); // Cách khai báo props trong Svelte 5+
</script>

<main class="flex flex-col justify-center items-center mt-5 gap-3">
  <h1 class="text-2xl">Todo</h1>
  {#await data.todos}
    <Loading />
  {:then todo}
    <h2>{todo.title}</h2>
  {:catch error}
    <p>{error.message}</p>
  {/await}
</main>
```

Svelte là framework yêu thích của cá nhân tôi cho đến nay, vì vậy tôi rất vui khi tính năng này có sẵn!

*   [Demo: Vercel](https://sveltekit-streaming-todo.vercel.app/)
*   [Kho mã nguồn: GitHub](https://github.com/jonathangamble/sveltekit-streaming-todo)
*   [Tài liệu: Streaming with Promises](https://kit.svelte.dev/docs/server-data#streaming-with-promises)

### Qwik

Qwik tuân theo cùng một mẫu, nhưng chưa sẵn sàng hoàn toàn.

**Loader**

```typescript
import { routeLoader$ } from '@builder.io/qwik-city';

type Todo = {
  title: string
};

export const useTodo = routeLoader$(() => {
  return async () => { // Trả về một Promise
    const randomTodo = Math.floor(Math.random() * 200) + 1;
    return await fetch(`https://jsonplaceholder.typicode.com/todos/${randomTodo}`)
      .then(r => r.json()) as Todo;
  }
});
```

**Component**

Tương tự như `<Suspense />`, Qwik có một component `<Resource />` để tải dữ liệu như mong đợi.

```tsx
import { component$ } from '@builder.io/qwik';
import { Resource } from '@builder.io/qwik';
import { useTodo } from './layout'; // Import loader
import Loading from '~/components/loading/loading'; // Cần cập nhật đường dẫn

export default component$(() => {
  const todo = useTodo(); // Lấy Resource

  return (
    <main class="flex flex-col justify-center items-center mt-5 gap-3">
      <h1 class="text-2xl">Todo</h1>
      <Resource
        value={todo}
        onPending={() => <Loading />} // onPending hiện chưa hoạt động đúng
        onResolved={(todo) => <h2>{todo.title}</h2>}
        // onRejected không được đề cập trong ví dụ gốc, nhưng Resource có
      />
    </main>
  );
});
```

*Lưu ý*: Hiện tại, `onPending` không hoạt động đúng để hiển thị trạng thái đang tải, và có một [GitHub Issue](https://github.com/BuilderIO/qwik/issues/3889) về vấn đề này. Qwik về mặt kỹ thuật truyền phát phản hồi, nhưng vẫn đợi nó hoàn thành. Phiên bản V2 sẽ khắc phục điều này.

### SolidStart

Framework máy chủ của SolidJS được xây dựng cho streaming, theo đúng nghĩa đen.

**Query Loader**

Truy vấn phải được tải với `'use server'` để hoạt động đúng.

```typescript
import { query } from '@solidjs/router'; // Import query
import { RouteDefinition } from "@solidjs/router"; // Import RouteDefinition

type Todo = {
  title: string
};

const getTodo = query(async () => {
  'use server'; // Cần thiết cho server query
  const randomTodo = Math.floor(Math.random() * 200) + 1;
  const todo = await fetch(`https://jsonplaceholder.typicode.com/todos/${randomTodo}`);
  return await todo.json() as Todo;
}, 'todo'); // Key cho query

export const route = {
  preload: () => getTodo() // Thêm function vào preload
} satisfies RouteDefinition;
```

Và hàm phải được thêm vào `preload`.

**Component**

```tsx
import { createAsync } from "@solidjs/router"; // Import createAsync
import { Suspense, Show } from "solid-js"; // Import Suspense, Show
import { ErrorBoundary } from "solid-js/web"; // Import ErrorBoundary
import { getTodo } from "./route"; // Import query loader
import Loading from "~/components/loading"; // Cần cập nhật đường dẫn

export default function Home() {
  const todo = createAsync(() => getTodo()); // Lấy dữ liệu bằng createAsync

  return (
    <main class="flex flex-col justify-center items-center mt-5 gap-3">
      <h1 class="text-2xl">Todo</h1>
      <ErrorBoundary fallback={<div>Đã xảy ra lỗi!</div>}> {/* Xử lý lỗi */}
        <Suspense fallback={<Loading />}> {/* Hiển thị loading */}
          <Show when={todo()}> {/* Hiển thị khi dữ liệu có */}
            {(data) => ( // Truy cập dữ liệu bằng hàm
              <h2>{data().title}</h2>
            )}
          </Show>
        </Suspense>
      </ErrorBoundary>
    </main>
  );
}
```

Chúng ta phải lấy dữ liệu bằng `createAsync` trước. Chúng ta có thể đặt nó bên trong `ErrorBoundary` để bắt lỗi. `Suspense` xác định khi nào chúng ta đang tải, và `Show` sẽ hiển thị component khi dữ liệu có sẵn.

Tôi thấy SolidJS có quá nhiều 'boilerplate' đối với sở thích cá nhân, nhưng nó đã hoạt động với Streaming ngay từ đầu.

*   [Demo: Vercel](https://solidstart-streaming-todo.vercel.app/)
*   [Kho mã nguồn: GitHub](https://github.com/jonathangamble/solidstart-streaming-todo)
*   [Tài liệu: Data loading always on the server](https://docs.solidjs.com/solid-start/how-it-works/data-loading)

### Nuxt

Nuxt hiện tại có một issue đang mở và kế hoạch thêm tính năng này sớm!

### Angular/Analog

Angular có lẽ sẽ không bao giờ triển khai tính năng này, ngay cả khi nó là framework bán máy chủ với `@angular/ssr`. Đã có một yêu cầu tính năng cũ bị từ chối. Analog có yêu cầu tính năng mới hơn, nhưng chưa có xác nhận.

## Tổng kết

Streaming là một công cụ tuyệt vời để thêm vào bộ công cụ của bạn. Khi bạn cần tăng tốc trang, nó chắc chắn sẽ giúp bạn vượt trội hơn đối thủ cạnh tranh với tốc độ tải trang SEO tốt hơn.

---

**Nguồn gốc:** Dựa trên bài viết "[Fantastic Framework Server Streaming](https://jonathangamble.com/fantastic-framework-server-streaming)" của Jonathan Gamble.
```

**Lưu ý:**

*   Tôi đã dịch các đoạn văn, tiêu đề và chú thích sang tiếng Việt.
*   Mã nguồn (code snippets) được giữ nguyên định dạng ban đầu như trong bài viết, vì đây là mã kỹ thuật và việc dịch tên biến hoặc cú pháp là không phù hợp. Tôi chỉ thêm các chú thích tiếng Việt giải thích các phần quan trọng.
*   Các tên kỹ thuật như "streaming", "Server Components", "loaders", "Suspense", "Resource", "ErrorBoundary", "First Contentful Paint", "Largest Contentful Paint", "Time to First Byte" được giữ nguyên hoặc cung cấp thêm giải thích tiếng Việt trong ngoặc đơn để đảm bảo độ chính xác và quen thuộc trong cộng đồng kỹ thuật.
*   Đường dẫn URL đến demo, repo và tài liệu được giữ nguyên.
*   Thông tin tác giả và ngày xuất bản được giữ lại.
*   Thêm một dòng nhỏ ở cuối để ghi rõ nguồn gốc bài viết.
*   Cập nhật cú pháp Svelte component cho Svelte 5+ `$props()` như trong code ví dụ, đồng thời sửa `PageProps` thành `PageData` theo quy ước SvelteKit.
*   Cập nhật đường dẫn import `Loading` cho Qwik và SolidStart để trông giống một dự án thực tế hơn (sử dụng alias `~/`).