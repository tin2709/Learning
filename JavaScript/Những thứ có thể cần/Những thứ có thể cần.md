
# 1  Xây dựng ứng dụng web chạy trên máy chủ với htmx: Tutorial To-Do App
Sponsor by https://blog.logrocket.com/htmx-server-driven-web-apps/?ref=dailydev

Dự án này là mã nguồn và giải thích dựa trên một hướng dẫn về cách sử dụng thư viện htmx để xây dựng một ứng dụng To-Do đơn giản, chạy trên máy chủ. Nó thể hiện phương pháp tiếp cận "hypermedia-driven" của htmx như một giải pháp thay thế cho các framework JavaScript client-side phức tạp trong một số trường hợp.

## Giới thiệu

Trong bối cảnh phát triển web hiện đại đang dịch chuyển nhanh chóng với các giải pháp phức tạp như Server Components hay các framework lớn (React, Vue, Svelte), sự đơn giản của các thư viện cổ điển như jQuery dường như ít phổ biến hơn. htmx ra đời nhằm thu hẹp khoảng cách này, mang đến cách tiếp cận đơn giản hóa để xây dựng các ứng dụng web "chạy trên máy chủ" (server-driven).

## htmx là gì?

*   **Thư viện JavaScript nhỏ, hướng trình duyệt:** htmx là một thư viện JavaScript gọn nhẹ (~14KB nén gzip), không phụ thuộc.
*   **Mục tiêu:** Xây dựng các ứng dụng web đơn giản, không rườm rà mà logic chính nằm ở phía máy chủ.
*   **Cách hoạt động cốt lõi:** Cho phép bạn thực hiện các yêu cầu AJAX (và nhiều hơn nữa) trực tiếp từ các phần tử HTML thông qua các **thuộc tính tùy chỉnh** (custom attributes).
*   **Phương pháp Hypermedia-driven:** Thay vì trao đổi dữ liệu (JSON, GraphQL) và xử lý UI hoàn toàn ở client, htmx khuyến khích máy chủ trả về các mảnh **siêu phương tiện** (thường là HTML) để client (htmx) chèn hoặc thay thế vào DOM hiện có.
*   **Không phải là phiên bản mới của HTML:** htmx là thư viện **bổ sung** sức mạnh cho HTML, giúp HTML có khả năng tương tác động với backend một cách dễ dàng.

### Ví dụ cơ bản

```html
<button
  hx-get="/path/to/api"
  hx-swap="innerHTML"
  hx-target="#target-container"
>
  Nhấn vào đây
</button>

<div id="target-container">Nội dung ban đầu...</div>
```
Khi nhấn nút, htmx sẽ gửi yêu cầu GET đến `/path/to/api`. Nội dung HTML nhận được từ phản hồi sẽ thay thế `innerHTML` của phần tử có `id="target-container"`.

## Ưu điểm của htmx

*   **Đơn giản:** Không yêu cầu kỹ năng JavaScript nâng cao.
*   **Nhẹ:** Kích thước nhỏ, không phụ thuộc.
*   **Cập nhật UI động:** Sử dụng AJAX để cập nhật một phần trang mà không cần tải lại toàn bộ.
*   **Ưu tiên Server-side:** Giữ logic template và xử lý chính ở máy chủ.
*   **Ít JavaScript ở Client:** Giảm đáng kể lượng mã JS cần viết thủ công.
*   **Nâng cao dần (Progressive Enhancement):** Ứng dụng cơ bản hoạt động với HTML tĩnh, htmx thêm lớp tương tác động.
*   **Hỗ trợ trình duyệt:** Hoạt động trên mọi trình duyệt hỗ trợ AJAX (bao gồm IE 11).
*   **Mở rộng:** Có các extension chính thức và cộng đồng để mở rộng chức năng.

## Nhược điểm của htmx

*   **Phụ thuộc vào Server:** Rất tập trung vào giao tiếp server. Các tác vụ client-side phức tạp (caching, state management cục bộ) cần giải pháp khác.
*   **Đường cong học tập:** Khác biệt với cách làm AJAX truyền thống hoặc framework JS hiện đại.
*   **Debugging:** Chưa có công cụ debugging tích hợp mạnh mẽ như framework lớn.
*   **Thao tác DOM phức tạp:** Các thao tác DOM phức tạp hoặc tùy chỉnh cao có thể vẫn cần JS thuần.
*   **Trải nghiệm cho Dev JS Framework:** Thiếu các khái niệm quen thuộc như component, state management, build tool phức tạp...
*   **Hệ sinh thái nhỏ:** Cộng đồng và thư viện/tooling xung quanh nhỏ hơn so với các framework lớn.
*   **Khả năng mở rộng (Scalability):** Có thể không phù hợp hoặc khó quản lý hơn cho các ứng dụng quy mô rất lớn, phức tạp về UI client-side.

## Có gì mới trong htmx 2.0+

*   API cải tiến.
*   Hỗ trợ các hệ thống module (ESM, AMD, CJS).
*   Xử lý sự kiện đơn giản hơn (`hx-on:`).
*   Hỗ trợ tốt hơn cho Web Components và Shadow DOM.

## Bắt đầu với htmx (Ví dụ To-Do App)

Chúng ta sẽ xây dựng một ứng dụng To-Do đơn giản sử dụng htmx, Express (backend) và Supabase (database).

### Yêu cầu

*   Hiểu biết cơ bản về HTML, JavaScript, HTTP requests, và Web nói chung.
*   Node.js và pnpm (hoặc npm/yarn) đã cài đặt.
*   Một tài khoản Supabase (free tier là đủ).

### Cài đặt và Cấu hình

1.  **Tạo thư mục dự án:**
    ```bash
    mkdir htmx-todo
    cd htmx-todo
    pnpm init -y
    ```

2.  **Cài đặt dependencies:**
    ```bash
    pnpm add htmx.org@2.0.4 express body-parser ejs @supabase/supabase-js
    ```

3.  **Cấu hình Supabase:**

    ![alt text](image.png)
    *   Tạo một dự án mới trên Supabase.
    *   Lấy `Project URL` và `anon key` từ phần Settings -> API.
    *   Tạo file `.env` ở thư mục gốc dự án và thêm:
        ```env
        SUPABASE_URL=<Project URL của bạn>
        SUPABASE_KEY=<Anon key của bạn>
        SERVER_HOST=localhost
        SERVER_PORT=5555 # Hoặc cổng nào bạn muốn
        ```
    *   **RẤT QUAN TRỌNG:** Thêm `.env` vào file `.gitignore` để tránh đẩy lên public repo.
    *   Tạo bảng `todos` trong Supabase bằng SQL Editor:
        ```sql
        CREATE TABLE todos (
          id SERIAL PRIMARY KEY,
          task TEXT NOT NULL,
          completed BOOLEAN DEFAULT FALSE,
          created_at TIMESTAMPTZ DEFAULT NOW()
        );
        ```

4.  **Tạo file `server.js`:**
    ```javascript
    // server.js
    import path from 'path';
    import { fileURLToPath } from 'url';
    import express from 'express';
    import bodyParser from 'body-parser';
    import { createClient } from '@supabase/supabase-js';

    const __filename = fileURLToPath(import.meta.url);
    const __dirname = path.dirname(__filename);

    // Lấy biến môi trường (đảm bảo chạy với --env-file=.env)
    const port = process.env.SERVER_PORT || 3000;
    const host = process.env.SERVER_HOST || 'localhost';
    const supabaseUrl = process.env.SUPABASE_URL;
    const supabaseKey = process.env.SUPABASE_KEY;

    const supabase = createClient(supabaseUrl, supabaseKey);

    const app = express();

    // Cấu hình Express
    app.set('view engine', 'ejs'); // Sử dụng EJS cho template
    app.use(bodyParser.urlencoded({ extended: true })); // Parse form data

    // Phục vụ file htmx từ node_modules
    app.get('/js/htmx.min.js', (req, res) => {
      res.sendFile(path.join(__dirname, 'node_modules/htmx.org/dist/htmx.min.js'));
    });

    // --- Các route API CRUD sẽ được thêm vào đây ---

    // Route gốc (để render trang chính và đọc dữ liệu)
    app.get('/', async (req, res) => {
      try {
        const { data: todos, error } = await supabase
          .from('todos')
          .select('*')
          .order('created_at', { ascending: false });

        if (error) throw error;

        res.render('index', {
          todos: todos || [],
          error: null,
        });
      } catch (error) {
        console.error('Error fetching todos:', error);
        res.render('index', {
          todos: [],
          error: 'Failed to load todos',
        });
      }
    });

    // Lắng nghe kết nối
    app.listen(port, () => {
      console.log(`Server running on http://${host}:${port}`);
    });
    ```

5.  **Cập nhật `package.json`:**
    ```json
    {
      "name": "htmx-todo",
      "...": "...",
      "type": "module", // Thêm dòng này để dùng import
      "scripts": {
        "start": "node --env-file=.env server.js",
        "dev": "node --env-file=.env --watch server.js"
      },
      "...": "..."
    }
    ```

6.  **Tạo thư mục `views` và các template EJS:**
    *   Tạo thư mục `views` ở gốc dự án.
    *   Tạo thư mục `partials` bên trong `views`.
    *   Tạo file `views/index.ejs`:
        ```html
        <!DOCTYPE html>
        <html>
          <head>
            <title>Todo App w/ htmx</title>
            <meta charset="utf-8" />
            <meta name="viewport" content="width=device-width" />
            <!-- Sử dụng CDN Tailwind CSS cho đơn giản, hoặc tự thêm CSS của bạn -->
            <script src="https://cdn.tailwindcss.com"></script> 
            <script src="/js/htmx.min.js"></script>
          </head>
          <body class="container mx-auto p-4">
            <h1 class="text-2xl font-bold mb-4">Ứng dụng To-Do với htmx</h1>

            <!-- Form thêm Todo -->
            <form
              hx-post="/todos"
              hx-target="#todo-list"
              hx-swap="outerHTML transition:true"
              hx-on::after-request="this.reset()"
              class="mb-4 flex gap-2"
            >
              <input
                type="text"
                name="task"
                required
                placeholder="Thêm công việc mới..."
                class="border p-2 flex-grow"
              />
              <button type="submit" class="bg-blue-500 text-white p-2 rounded">
                Thêm
              </button>
            </form>

            <!-- Container chứa danh sách Todo -->
            <% if (error) { %>
              <%- include('partials/error', { message: error }) %>
            <% } else{ %>
              <%- include('partials/todo-list', { todos }) %>
            <% } %>

          </body>
        </html>
        ```
    *   Tạo file `views/partials/todo-item.ejs`:
        ```html
        <li id="todo-<%= todo.id %>" class="flex items-center justify-between border-b p-2">
          <form
            hx-put="/todos/<%= todo.id %>"
            hx-target="#todo-<%= todo.id %>"
            hx-swap="outerHTML"
            hx-on:change="this.requestSubmit()" 
            class="flex items-center flex-grow gap-2"
          >
            <!-- Input checkbox để cập nhật trạng thái -->
            <!-- `name="completed"` quan trọng để backend nhận được giá trị -->
            <input 
              type="checkbox" 
              id="todo-item-<%= todo.id %>" 
              name="completed" 
              <%= todo.completed ? 'checked' : '' %>
              class="form-checkbox h-5 w-5 text-blue-600"
            />
            <!-- Input hidden để gửi task (nếu cần cập nhật task) -->
            <input
              type="hidden"
              name="task"
              value="<%= todo.task %>"
            />
            <!-- Label hiển thị task, gạch ngang nếu hoàn thành -->
            <label
              for="todo-item-<%= todo.id %>"
              class="cursor-pointer flex-grow"
              style="text-decoration: <%= todo.completed ? 'line-through' : 'none' %>"
            >
              <%= todo.task %>
            </label>
          </form>
          <!-- Nút xóa -->
          <button
            type="button"
            hx-delete="/todos/<%= todo.id %>"
            hx-target="#todo-list"
            hx-swap="outerHTML transition:true"
            hx-confirm="Bạn có chắc muốn xóa công việc này?"
            class="bg-red-500 text-white p-1 rounded text-sm"
          >
            Xóa
          </button>
        </li>
        ```
    *   Tạo file `views/partials/todo-list.ejs`:
        ```html
        <div id="todo-list">
          <% if (todos && todos.length > 0) { %>
          <ul>
            <% [...todos].forEach(todo => { %> 
              <%- include('todo-item', { todo }) %> 
            <% }) %>
          </ul>
          <% } else { %>
          <div class="text-center text-gray-500">
            <p>Chưa có công việc nào!</p>
          </div>
          <% } %>
        </div>
        ```
    *   Tạo file `views/partials/error.ejs`:
        ```html
        <div class="text-red-500 bg-red-100 border border-red-400 p-3 rounded mb-4">
          <p><%= message %></p>
        </div>
        ```

### Implement API CRUD (Thêm vào `server.js`)

Thêm các route sau vào file `server.js` của bạn, trước phần `app.listen`.

**Tạo (Create - POST /todos):**

```javascript
// server.js
// ... (các import và setup khác) ...

app.post('/todos', async (req, res) => {
  try {
    const { task } = req.body;

    if (!task || task.trim().length === 0) {
      return res.render('partials/error', { message: 'Task là bắt buộc' });
    }

    // Thêm vào database
    const { data: newTodo, error: insertError } = await supabase
      .from('todos')
      .insert([{ task: task.trim() }])
      .select() // Chọn bản ghi vừa thêm để trả về
      .single(); // Lấy một bản ghi duy nhất nếu thành công

    if (insertError) throw insertError;

    // Sau khi thêm, lấy lại toàn bộ danh sách để render lại phần `todo-list`
    const { data: todos, error: fetchError} = await supabase
      .from('todos')
      .select('*')
      .order('created_at', { ascending: false });

    if (fetchError) throw fetchError;

    // Trả về template `todo-list` mới
    res.render('partials/todo-list', { todos });
  } catch (error) {
    console.error('Error creating todo:', error);
    // Trả về template lỗi
    res.render('partials/error', { message: `Lỗi khi thêm: ${error.message || error}` });
  }
});

// ... (các route khác và app.listen) ...
```
*Lưu ý:* Đoạn code gốc trong bài viết chỉ insert rồi fetch lại toàn bộ list. Cách hiệu quả hơn với htmx là chỉ insert và trả về *item* vừa tạo, sau đó dùng `hx-swap="afterbegin"` trên list. Tuy nhiên, theo sát bài viết gốc, chúng ta sẽ fetch và render lại toàn bộ list.

**Đọc (Read - GET /):**

Route này đã có sẵn trong file `server.js` ở phần cài đặt. Nó fetch toàn bộ todos khi trang được tải lần đầu và render `index.ejs`, bao gồm cả `todo-list.ejs`.

**Cập nhật (Update - PUT /todos/:id):**

```javascript
// server.js
// ... (các route khác) ...

app.put('/todos/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { task, completed } = req.body; // completed sẽ là 'on' hoặc undefined/''

    // Tạo object cập nhật
    const updates = {
      // Cập nhật task nếu có (không cần trong ví dụ này, nhưng để phòng)
      task: task?.trim(), 
      // completed sẽ là true nếu checkbox được check ('on'), false nếu không
      completed: completed === 'on' 
    };

    // Xóa các giá trị undefined khỏi object cập nhật
    Object.keys(updates).forEach(key => 
      updates[key] === undefined && delete updates[key]
    );

    // Cập nhật trong database
    const { error: updateError } = await supabase
      .from('todos')
      .update(updates)
      .eq('id', id);

    if (updateError) throw updateError;

    // Lấy lại bản ghi todo vừa cập nhật để render lại item đó
    const { data: todo, error: fetchError } = await supabase
      .from('todos')
      .select('*')
      .eq('id', id)
      .single();

    if (fetchError) throw fetchError;

    // Trả về template của một item todo
    res.render('partials/todo-item', { todo });
  } catch (error) {
    console.error('Error updating todo:', error);
    res.render('partials/error', { message: 'Lỗi khi cập nhật công việc' });
  }
});

// ... (các route khác và app.listen) ...
```
Lưu ý: Trong template `todo-item.ejs`, chúng ta dùng `hx-on:change="this.requestSubmit()"` trên form. Khi checkbox thay đổi trạng thái, form sẽ tự động submit yêu cầu PUT.

**Xóa (Delete - DELETE /todos/:id):**

```javascript
// server.js
// ... (các route khác) ...

app.delete('/todos/:id', async (req, res) => {
  try {
    const { id } = req.params;

    // Xóa khỏi database
    const { error: deleteError } = await supabase
      .from('todos')
      .delete()
      .eq('id', id);

    if (deleteError) throw deleteError;

    // Lấy lại toàn bộ danh sách còn lại để render lại phần list
    const { data: remainingTodos, error: fetchError } = await supabase
      .from('todos').select('*').order('created_at', { ascending: false });

    if (fetchError) throw fetchError;

    // Trả về template `todo-list` mới.
    // htmx sẽ tự xóa item bị delete nhờ hx-target="#todo-list" và hx-swap="outerHTML".
    // Tuy nhiên, bài viết gốc dùng HX-Retarget, làm cho logic hơi khác.
    // Theo bài viết gốc:
    // res.setHeader('HX-Retarget', '#todo-list'); // Yêu cầu htmx render kết quả vào #todo-list
    res.render('partials/todo-list', { todos: remainingTodos });

    // Cách khác theo htmx thông thường: Nếu hx-target là item bị xóa,
    // trả về response rỗng (hoặc mã 200) để htmx tự xóa phần tử đó.
    // Ví dụ: res.send('') hoặc res.status(200).send('')
    // Nhưng vì hx-target trong template delete button là #todo-list,
    // cách trả về todo-list mới như trên là hợp lý hơn theo bài viết gốc.

  } catch (error) {
    console.error('Error deleting todo:', error);
    res.render('partials/error', { message: 'Lỗi khi xóa công việc' });
  }
});

// ... (app.listen) ...
```
Lưu ý: Bài viết gốc đề cập đến header `HX-Retarget`. Tuy nhiên, với cấu hình `hx-target="#todo-list"` và `hx-swap="outerHTML"` trên nút xóa, việc trả về template `todo-list` mới chứa các item còn lại là đủ để htmx thay thế toàn bộ phần `#todo-list` bằng nội dung mới. Header `HX-Retarget` có thể dùng trong các trường hợp phức tạp hơn khi kết quả trả về cần được chèn vào một nơi khác với `hx-target` đã chỉ định ban đầu.

### Chạy ứng dụng

Mở terminal trong thư mục gốc dự án và chạy:
```bash
pnpm dev
```
Ứng dụng sẽ chạy trên địa chỉ và cổng đã cấu hình trong `.env` (ví dụ: `http://localhost:5555`).

![alt text](image-1.png)

## Các thuộc tính htmx chính được sử dụng

*   `hx-get`, `hx-post`, `hx-put`, `hx-delete`: Chỉ định phương thức HTTP và URL cho yêu cầu AJAX.
*   `hx-target`: Chỉ định phần tử nào trên trang sẽ được cập nhật với nội dung từ phản hồi.
*   `hx-swap`: Chỉ định cách nội dung mới sẽ thay thế nội dung cũ trong phần tử mục tiêu (ví dụ: `innerHTML`, `outerHTML`, `transition:true`).
*   `hx-trigger`: Chỉ định sự kiện nào sẽ kích hoạt yêu cầu (ví dụ: `load`, `click`, `change`). Mặc định là `click` cho hầu hết các phần tử, `change` cho input/select/textarea, `submit` cho form.
*   `hx-confirm`: Hiển thị hộp thoại xác nhận trước khi gửi yêu cầu.
*   `hx-on:`: Bắt các sự kiện của htmx (ví dụ: `hx-on::after-request` để reset form).

## Kết luận
Video: https://blog.logrocket.com/wp-content/uploads/2024/05/2025-04-24-01-49-26-1.mp4?_=1

htmx mang đến một cách tiếp cận khác biệt so với các framework client-side hiện đại. Nó phù hợp cho các ứng dụng nhỏ đến trung bình, nơi mà việc giữ logic render và xử lý chính ở phía server là ưu tiên, và cần thêm các tương tác động đơn giản mà không muốn nhúng sâu vào thế giới JavaScript framework phức tạp. Nếu bạn quen với việc xây dựng ứng dụng dựa trên server-rendered HTML và muốn thêm các chức năng AJAX một cách dễ dàng, htmx là một lựa chọn đáng xem xét.

Hãy thử nghiệm thêm với các tính năng khác của htmx và các extension của nó!

## Dựa trên

Bài viết này dựa trên hướng dẫn "creating server-driven web apps with htmx" (nguồn không được cung cấp trong văn bản gốc).
```

**Cách sử dụng:**

1.  Lưu nội dung trên vào một file tên `README.md` trong thư mục gốc của dự án.
2.  Thực hiện các bước cài đặt và code theo hướng dẫn trong file README.
3.  Sử dụng Markdown viewer/editor để xem file README đã định dạng.

Chúc bạn thành công!