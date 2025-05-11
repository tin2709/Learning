

# 1 Tại sao bạn sẽ yêu Solid JS nếu là một React developer

**(Dựa trên bài viết từ MayFest2025)**

Bạn là một lập trình viên React và cảm thấy mệt mỏi với những vấn đề về hiệu năng, re-render không kiểm soát, hay khó khăn khi tích hợp các thư viện JavaScript thuần? Bạn nghe nói về Solid JS và tự hỏi liệu đây có phải chỉ là "lại một JS framework nữa"?

Bài viết này sẽ giúp bạn hiểu tại sao Solid JS đáng để bạn quan tâm, đặc biệt nếu bạn đã quen thuộc với React. Solid JS mang đến một giải pháp mạnh mẽ cho những điểm yếu phổ biến của React, với cú pháp cực kỳ quen thuộc, giúp bạn giải quyết các vấn đề đau đầu mà không cần phải học lại từ đầu.

## Những Vấn Đề Đau Đầu Thường Gặp Với React

Nếu bạn đã làm việc với React đủ lâu, có lẽ bạn đã từng gặp phải những tình huống sau:

*   **Component chạy chậm không rõ lý do:** Một component lớn bị re-render quá nhiều lần (ví dụ: khi cửa sổ trình duyệt resize, kéo thả, form có nhiều trường), dẫn đến lag. Việc tìm ra nguyên nhân và fix thường phức tạp và rủi ro.
    *   *Ví dụ:* Xây dựng một component có thể resize chứa nội dung nặng, bạn có thể gặp khó khăn với việc cập nhật state quá thường xuyên, buộc phải chuyển sang dùng `ref` để cải thiện hiệu năng.
*   **Tích hợp thư viện Vanilla JS khó khăn:** Việc đưa các thư viện JavaScript thuần không được thiết kế riêng cho React (như D3, GSAP, VisJS Timeline) vào ứng dụng React đôi khi rất chật vật. Bạn phải vật lộn với `useRef`, `useEffect`, `useMemo`, `useCallback` và vẫn có thể gặp phải các vấn đề re-render không mong muốn.

Bạn ước gì có một framework giải quyết những vấn đề này mà không yêu cầu bạn phải học một "ngôn ngữ" hoàn toàn mới? Chào mừng bạn đến với Solid JS.

## Tại Sao Solid JS Là Lựa Chọn Tuyệt Vời Cho Developer React?

Solid JS được xây dựng để giải quyết trực tiếp những vấn đề trên, đồng thời tận dụng tối đa kinh nghiệm mà bạn đã có với React.

### 1. Cú Pháp Cực Kỳ Quen Thuộc

Bạn không có thời gian để học một framework hoàn toàn mới? Đừng lo! Cú pháp của Solid JS rất giống với React.
*   `useState` trong React tương đương với `createSignal` trong Solid.
*   `useEffect` trong React tương đương với `createEffect` trong Solid.

```javascript
// React Counter Example
import React, { useState } from 'react';

const App = () => {
  const [count, setCount] = useState(1);

  return (
    <div>
      <div>{count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
};
```

```javascript
// Solid JS Counter Example
import { createSignal } from "solid-js";

const App = () => {
  const [count, setCount] = createSignal(1); // Use createSignal

  return (
    <div>
      <div>{count()}</div> {/* Access signal value with () */}
      <button onClick={() => setCount(count() + 1)}> {/* Update signal with () */}
        Increment
      </button>
    </div>
  );
};
```

**Lưu ý nhỏ:** Trong Solid, bạn truy cập giá trị của signal bằng cách gọi nó như một hàm (`count()`) và cập nhật nó bằng cách gọi setter (`setCount(...)`). Ngoài ra, hãy tránh destructuring props trực tiếp như trong React.

```javascript
// NOT OK in Solid!
const Component = ({ data, onClick }) => {}

// OK in Solid!
const Component = (props) => {
	props.data;
	props.onClick();
}
```

Với chỉ vài khác biệt nhỏ này, bạn gần như có thể bắt đầu code Solid JS ngay lập tức!

### 2. Không Cần Lo Lắng Về Re-render Thừa Thãi

Solid JS có một cơ chế phản ứng (reactivity) gọi là "fine-grained reactivity". Thay vì re-render toàn bộ cây component khi state thay đổi (như React thường làm), Solid JS chỉ cập nhật trực tiếp những phần UI *thực sự* bị ảnh hưởng bởi sự thay đổi state.

Điều này có nghĩa là bạn có thể đặt mọi thứ vào state (signal) mà không phải lo lắng về hiệu năng. Ví dụ component resize chậm trong React khi dùng `useState` sẽ chạy mượt mà trong Solid JS dù bạn dùng `createSignal` tương tự:

```javascript
// Solid JS Resizable Component (Similar syntax to React, but fast!)
import { createSignal } from "solid-js";

const ResizableComponent = () => {
  const [width, setWidth] = createSignal(100); // Use signal, no need for ref just for performance

  let containerRef; // Use variable for ref

  const onDrag = (e) => {
    if (containerRef) {
      const delta = e.pageX - containerRef.getBoundingClientRect().right;
      setWidth(width() + delta); // Update signal
    }
  };

  return (
    <div
      ref={containerRef} // Assign ref
      style={{
        width: `${width()}px`, // Access signal value
      }}
    >
      <ExpensiveComponent /> {/* This won't re-render excessively */}

      {/* Handle */}
      <div
        onMouseDown={() => {
          document.addEventListener("mousemove", onDrag);
          document.addEventListener("mouseup", () => {
            document.removeEventListener("mousemove", onDrag);
          });
        }}
      />
    </div>
  );
};
```
(Lưu ý: `ExpensiveComponent` ở đây giả định là một component nặng không có logic Solid cụ thể cần re-render).

Với Solid JS, bạn có thể quên đi `useRef` (cho mục đích tối ưu hiệu năng), `useCallback`, `useMemo` - bạn chỉ cần tập trung vào logic của mình và Solid JS sẽ đảm bảo mọi thứ hoạt động nhanh chóng.

### 3. Tích Hợp Dễ Dàng Với Thư Viện JavaScript Thuần

Vì Solid JS chỉ mount mỗi component một lần và cập nhật DOM trực tiếp dựa trên thay đổi của signal (thay vì re-render toàn bộ component), việc tích hợp các thư viện vanilla JS trở nên cực kỳ đơn giản.

Bạn chỉ cần:
1.  Tạo một `div` hoặc element HTML khác làm container.
2.  Sử dụng thuộc tính `ref` để lấy tham chiếu đến element đó.
3.  Sử dụng hook `onMount` (tương tự `useEffect` với dependency array rỗng trong React) để chạy code vanilla JS khi component được mount.
4.  Truyền tham chiếu (`ref`) đó vào thư viện vanilla JS của bạn.

```javascript
// Example: Integrating Vis-Timeline in Solid JS
import { onMount } from "solid-js";
import { Timeline } from "vis-timeline";
import "vis-timeline/dist/vis-timeline-graph2d.min.css";

const VisTimelineDemo = () => {
  let containerRef; // Ref variable

  onMount(() => { // Run code when component mounts
    const items = [ /* ... your timeline items ... */ ];
    const timeline = new Timeline(containerRef, items, {}); // Use the ref
    // ... further timeline manipulation ...
  });

  return (
    <div
      ref={containerRef} // Assign the ref
      style={{ width: '600px', height: '400px' }}
    />
  );
};
```

Code tích hợp D3 hay các thư viện tương tự cũng đơn giản không kém. Bạn có thể kết hợp cách viết component quen thuộc của React với sức mạnh và sự đơn giản của JavaScript thuần khi cần thiết.

## Quản Lý State: Context và Store Trong Solid JS

Solid JS cung cấp các cách tiếp cận đơn giản và hiệu quả để quản lý state trên toàn ứng dụng.

### Context Đơn Giản Hơn Bao Giờ Hết

Bạn có state cần chia sẻ giữa các component ở các cấp độ khác nhau? Trong Solid JS, bạn chỉ cần khai báo signal/state đó ở bên ngoài các component và import nó vào nơi bạn cần sử dụng.

```javascript
// Store file (e.g., store.js)
import { createSignal } from "solid-js";

export const [count, setCount] = createSignal(1);
```

```javascript
// ComponentA (uses count and can modify)
import { count, setCount } from "./store";

const ComponentA = () => {
  return (
    <div>
      <div>Count: {count()}</div>
      <button onClick={() => setCount(count() + 1)}>Increment</button>
    </div>
  );
};
```

```javascript
// ComponentB (uses count and can modify)
import { count, setCount } from "./store";

const ComponentB = () => {
  return (
    <div>
      <div>Count: {count()}</div>
      <button onClick={() => setCount(count() - 1)}>Decrement</button>
    </div>
  );
};
```

Không cần Context Provider, Consumer, `useContext`, hay lo lắng về performance khi Context thay đổi!

### Quản Lý Store Phức Tạp Với `createStore`

Đối với dữ liệu có cấu trúc phức tạp (như mảng object, nested data), Solid JS cung cấp `createStore`. Pattern này rất giống với cách bạn làm việc với Redux "slices".

```javascript
// Store file (e.g., todosStore.js)
import { createStore, produce } from "solid-js/store";

const [todos, setTodos] = createStore([]);

// Selector-like function
export const selectTodos = () => todos;

// Dispatcher-like function
export const addTodo = (newTodo) => {
  setTodos(produce(state => {
    state.push(newTodo);
  }));
};
```

```javascript
// Component sử dụng Store
import { selectTodos, addTodo } from "./todosStore";
import { createSignal, For } from "solid-js";

const AddTodo = () => {
  const [inputTodo, setInputTodo] = createSignal("");
  return (
    <>
      <input onInput={(e) => setInputTodo(e.currentTarget.value)} />
      <button onClick={() => addTodo(inputTodo())}>Add Todo</button>
    </>
  );
};

const TodoList = () => {
  const todos = selectTodos(); // Access store data

  return (
    <For each={todos}>
      {(todo) => <div>{todo}</div>}
    </For>
  );
};

// Có thể render cả hai component này ở đâu đó trong cây ứng dụng
```
Bạn có thể quản lý state phức tạp theo một pattern quen thuộc tương tự Redux, nhưng với cú pháp đơn giản hơn.

## Nhược Điểm Cần Cân Nhắc

Mặc dù có nhiều ưu điểm, Solid JS cũng có một số nhược điểm hiện tại:

*   **Hệ sinh thái nhỏ hơn:** Solid JS chưa phổ biến bằng React, nên có ít các thư viện chuyên dụng tương tự như `react-flow`, `react-three-fiber`, các thư viện input masking/validation phức tạp, v.v.
*   **Solid Start chưa trưởng thành bằng NextJS:** Nếu bạn cần Server-Side Rendering (SSR), Solid Start là lựa chọn của Solid JS, nhưng nó chưa "battle-tested" và có cộng đồng lớn mạnh như NextJS.

## Kết Luận

Nếu bạn là một lập trình viên React và đang tìm kiếm một framework mới có thể giải quyết các vấn đề về hiệu năng và tích hợp thư viện một cách dễ dàng, trong khi vẫn giữ lại cú pháp quen thuộc, Solid JS là một ứng cử viên sáng giá.

Lần đầu trải nghiệm Solid JS, bạn có thể cảm thấy ngạc nhiên về sự mượt mà và đơn giản của nó. Bạn có thể tập trung vào xây dựng tính năng mà không phải liên tục suy nghĩ về re-render, memoization hay cách "đưa" thư viện ngoài vào. Solid JS cho phép bạn kết hợp cách viết component hiện đại với sự linh hoạt của JavaScript thuần.

Solid JS có tiềm năng rất lớn. Nếu bạn có một dự án cá nhân, một MVP cần tốc độ phát triển và hiệu năng cao, hoặc muốn thử nghiệm một cách tiếp cận khác để giải quyết các vấn đề performance trong nhóm React của mình, hãy dành thời gian tìm hiểu và thử sức với Solid JS!

