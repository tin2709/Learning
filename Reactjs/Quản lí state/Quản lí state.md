

# 1  Quản lý State trong React bằng MobX


Nếu bạn làm việc với React, việc quản lý state là một phần không thể thiếu. Khi ứng dụng trở nên phức tạp hơn, việc truyền dữ liệu giữa các component lồng nhau có thể trở thành một thách thức lớn. Bài viết này giới thiệu MobX, một thư viện quản lý state phổ biến và hiệu quả cho các ứng dụng React, được đánh giá là dễ tiếp cận hơn so với Redux đối với người mới bắt đầu.

**Lưu ý:** Nội dung này được viết lại từ các bài đăng cũ hơn (3 năm trước), có thể một số API hoặc best practice đã thay đổi trong các phiên bản MobX/React mới nhất.

## Tại Sao Cần Quản Lý State?

Trong React, việc truyền dữ liệu từ component cha xuống component con thường được thực hiện thông qua props. Với một component cha lớn chứa nhiều component con lồng sâu, bạn sẽ gặp phải tình trạng "props drilling" – phải truyền props qua nhiều cấp độ component trung gian không thực sự sử dụng dữ liệu đó. Điều này khiến code trở nên khó đọc, khó bảo trì và dễ gây lỗi.

Để giải quyết vấn đề này, các thư viện quản lý state như MobX và Redux ra đời, giúp tập trung state vào một nơi duy nhất (store) và cho phép các component truy cập state mà không cần truyền props qua lại.

## Tại Sao Lựa Chọn MobX?

Tác giả bài viết chọn giới thiệu MobX trước vì nhận thấy MobX có đường cong học tập (learning curve) thoải mái hơn so với Redux. MobX có những đặc điểm nổi bật:

*   **Đơn giản và dễ hiểu:** Cú pháp MobX thường ngắn gọn và trực quan.
*   **Mutable State:** MobX cho phép thay đổi state trực tiếp (mutation), khác với Redux thường yêu cầu immutable state. Điều này có thể cảm thấy quen thuộc và dễ tiếp cận hơn đối với một số developer.
*   **Scalable:** Phù hợp với các ứng dụng từ nhỏ đến lớn.

## MobX Là Gì?

MobX là một thư viện quản lý trạng thái (state management) độc lập. Nó giúp bạn định nghĩa dữ liệu (state) và cách các phần khác của ứng dụng (như UI React) tự động phản ứng với những thay đổi của dữ liệu đó.

## Những Điều Cần Lưu Ý Khi Dùng MobX

*   **Mutable:** MobX làm việc với state có thể thay đổi trực tiếp.
*   **Code ngắn gọn:** So với Redux, MobX thường yêu cầu ít boilerplate code hơn.

## MobX Trong React

MobX kết hợp rất tốt với React, đặc biệt là với Function Component (sử dụng `mobx-react-lite`). Dưới đây là các khái niệm và cách sử dụng cơ bản.

### 1. Observable

`observable` là trái tim của MobX. Nó biến một giá trị (primitives, objects, arrays, class instances, Map, Set,...) thành một giá trị có thể "quan sát" được. MobX sẽ theo dõi những giá trị `observable` này để biết khi nào state thay đổi và cần cập nhật những thứ phụ thuộc vào nó.

Bạn có thể tạo `observable` theo nhiều cách:

```javascript
import { observable } from 'mobx';

// Theo dõi một mảng
const list = observable([1, 2, 3, 4]);
list[2] = 5; // Thay đổi trực tiếp

// Theo dõi một property trong class instance (sử dụng decorator)
// Cần cấu hình Babel hoặc TypeScript để hỗ trợ decorator
class Todo {
  @observable title = 'Mua banh mi';
}
const TODO = new Todo();
TODO.title = 'Đã mua banh mi'; // Thay đổi trực tiếp

// Hoặc theo dõi một plain object
const person = observable({
  firstName: "Clive Staples",
  lastName: "Lewis",
});
person.firstName = "C.S."; // Thay đổi trực tiếp
```

Tất cả các thay đổi trên đều là thay đổi trực tiếp (mutate) vào object hoặc mảng `observable` ban đầu.

### 2. Observer

`observer` là một React Higher-Order Component (HOC), decorator, hoặc hook giúp component React có khả năng "quan sát" các giá trị `observable`. Khi bất kỳ `observable` nào mà component này truy cập bị thay đổi, component sẽ tự động re-render để hiển thị state mới nhất.

Có hai package chính:
*   `mobx-react`: Hỗ trợ cả Class Component và Function Component (API cũ hơn cho Function Component).
*   `mobx-react-lite`: Nhỏ hơn, được tối ưu cho Function Component (thường được khuyên dùng cho Function Component).

Các cách sử dụng `observer` trong Function Component:

```javascript
import { observable } from 'mobx';
import { Observer, useObserver, observer } from 'mobx-react-lite'; // Hoặc mobx-react cho class component

// Tạo một dữ liệu observable
const person = observable({
  name: 'John',
});

// Cách 1: observer HOC (Phổ biến nhất cho Function Component)
// Bọc toàn bộ function component bằng observer()
const P1 = observer(({ person }) => {
  return <h1>{person.name}</h1>; // Truy cập person.name
});

// Cách 2: Observer component
// Chỉ bọc phần UI cần re-render bên trong <Observer>
const P2 = ({ person }) => (
  <>
    {/* Phần này sẽ KHÔNG re-render khi person.name thay đổi nếu không được bọc */}
    <p>{person.name} dont want to change my name</p>
    {/* Phần này sẽ re-render */}
    <Observer>{() => <h1>{person.name} will change to new name</h1>}</Observer>
  </>
);

// Cách 3: useObserver hook
// Trả về một component mới đã được observer
const P3 = ({ person }) => {
  return useObserver(() => <h1>{person.name}</h1>); // Truy cập person.name bên trong useObserver
};

// Component cha render các component con
const DetailPerson = () => (
  <>
    <P1 person={person} />
    <P2 person={person} />
    <P3 person={person} />
  </>
);

// Thay đổi dữ liệu sau một khoảng thời gian
setTimeout(() => {
  person.name = 'Jane'; // Thay đổi observable state
}, 1000);

// Render DetailPerson (ví dụ với ReactDOM)
// ReactDOM.render(<DetailPerson />, document.getElementById('root'));
```
Khi `person.name` thay đổi, các component `P1`, `P3` và phần bên trong `<Observer>` của `P2` sẽ tự động re-render.

### 3. Cấu Trúc Khuyến Nghị Với React Context

Để quản lý state toàn cục hoặc state phức tạp hiệu quả, MobX Team khuyến nghị sử dụng React Context để cung cấp MobX store đến các component.

Bạn nên tạo một file riêng để định nghĩa store và các hành động (actions):

**`store.ts` (Ví dụ đơn giản)**

```typescript
import { createContext, useContext } from "react";
import { observable, action } from 'mobx';

// Sử dụng decorator @observable và @action requires cấu hình
class Todo {
  @observable title = 'Mua banh mi';

  // @action đánh dấu một phương thức/function là hành động làm thay đổi state.
  // MobX khuyến nghị dùng action cho các thay đổi state để có cấu trúc tốt hơn.
  // Có thể dùng action.bound để bind this tự động
  @action
  changeTitle = () => {
    this.title = 'Đã mua banh mi';
  };
}

// rootStore chứa tất cả các store con
export const rootStore = {
  todoStore: new Todo()
};

export type TRootStore = typeof rootStore;

// Tạo React Context
const RootStoreContext = createContext<null | TRootStore>(null);

// Provider component để bọc ứng dụng và cung cấp store
export const Provider = RootStoreContext.Provider;

/**
 * Custom hook để truy cập store trong Function Component.
 * Throws error nếu không có Provider bọc ngoài.
 */
export function useStore() {
  const store = useContext(RootStoreContext);
  if (store === null) {
    throw new Error("Store cannot be null, please add a context provider");
  }
  return store;
}
```

**`index.tsx` (Entry point của ứng dụng)**

```typescript
import * as React from "react";
import { render } from "react-dom";
import { Provider, rootStore } from "./store"; // Import Provider và rootStore
import { App } from "./App"; // Component gốc của bạn

const rootElement = document.getElementById("root");

/** Bọc ứng dụng bằng Provider và truyền rootStore vào value */
render(
  <Provider value={rootStore}>
    <App /> {/* App component và tất cả con cháu sẽ có thể truy cập store */}
  </Provider>,
  rootElement
);
```

**`Todo.tsx` (Component sử dụng store)**

```typescript
import { observer } from 'mobx-react-lite'; // Sử dụng observer cho Function Component
import React from 'react';

import { useStore } from './store'; // Import hook useStore

// Component Todo được bọc bởi observer
export const Todo = observer(() => {
  const { todoStore } = useStore(); // Lấy todoStore từ rootStore thông qua hook

  return (
    <>
      {/* Truy cập observable state title */}
      <p>title: {todoStore.title}</p>
      {/* Gọi action để thay đổi state */}
      <button onClick={todoStore.changeTitle}>Click to change title</button>
    </>
  );
});
```
Với cấu trúc này, bất kỳ component nào cũng có thể truy cập state từ store bằng cách sử dụng `useStore` hook (trong Function Component) mà không cần truyền props qua nhiều cấp.

## Các Hàm MobX Đặc Biệt Khác

MobX cung cấp thêm một số hàm tiện ích để làm việc với reactivity.

### 1. `useLocalStore` (Dành cho Local State)

`useLocalStore` (trong `mobx-react` hoặc `mobx-react-lite`) là một hook cho phép tạo một MobX store nhỏ, cục bộ bên trong Function Component. Nó có thể là một giải pháp thay thế cho `useState` khi bạn có nhiều state cục bộ liên quan đến nhau và muốn quản lý chúng theo kiểu store.

```javascript
import React from "react";
import { useLocalStore, observer } from "mobx-react-lite"; // hoặc mobx-react

export const CounterWithLocalStore = observer(() => {
  // Khai báo một store cục bộ
  const store = useLocalStore(() => ({
    count: 0,
    // Các hàm thay đổi state được định nghĩa trực tiếp trong store
    increase: () => (store.count += 1), // Mutation trực tiếp
    decrease: () => (store.count -= 1),
    reset: () => (store.count = 0),
  }));
  // Destructure các giá trị và hàm từ store để sử dụng trong JSX
  const { count, increase, decrease, reset } = store;

  return (
    <>
      <p>Count (with local store): {count}</p>
      <button onClick={increase}>Increase +</button>
      <button onClick={decrease}>Decrease -</button>
      <button onClick={reset}>Reset</button>
    </>
  );
});
```
Thay vì dùng `useState` riêng lẻ cho từng giá trị, bạn có thể nhóm chúng vào một local store và định nghĩa các hàm thay đổi ngay tại đó.

### 2. `computed`

`computed` dùng để định nghĩa các giá trị được "tính toán" từ các giá trị `observable` khác. Giá trị `computed` chỉ được tính toán lại khi các `observable` mà nó phụ thuộc thay đổi. Điều này giúp tối ưu hiệu năng bằng cách tránh các phép tính không cần thiết.

```javascript
import { observable, computed } from "mobx";

class OrderLine {
    @observable price = 0;
    @observable amount = 1;

    constructor(price) {
      this.price = price;
    }

    // @computed get định nghĩa một getter sẽ tự động tính toán lại
    // khi price hoặc amount thay đổi.
    @computed get total() {
      console.log("Calculating total..."); // Chỉ chạy khi price hoặc amount thay đổi
      return this.price * this.amount;
    }
}

const item = new OrderLine(10);
console.log(item.total); // "Calculating total..." -> 10

item.amount = 5; // Thay đổi amount
console.log(item.total); // "Calculating total..." -> 50 (Tính toán lại vì amount đổi)

item.amount = 5; // Giá trị không đổi
console.log(item.total); // 50 (Không tính toán lại vì amount không đổi)
```

### 3. `autorun`

`autorun` chạy một hàm (side effect) bất cứ khi nào bất kỳ `observable` nào được truy cập bên trong hàm đó thay đổi. Nó thường được dùng cho các side effect không tạo ra một giá trị mới (ví dụ: logging, đồng bộ hóa). `autorun` trả về một hàm `disposer` để dừng việc theo dõi.

```javascript
import { observable, computed, autorun } from "mobx";

class OrderLine {
    @observable price = 0;
    @observable amount = 1;

    constructor(price) {
        this.price = price;
        // autorun sẽ chạy khi total thay đổi
        const disposer = autorun((reaction) => {
          console.log(`Autorun: Total is ${this.total}`);
          // reaction.dispose() có thể dùng để dừng autorun theo điều kiện
          if (this.total > 10) {
            console.log("Total exceeded 10, disposing autorun.");
            reaction.dispose(); // Dừng việc theo dõi
          }
        });
        // Có thể gọi disposer() bất cứ lúc nào để dừng autorun
        // setTimeout(() => disposer(), 5000); // Ví dụ: dừng sau 5 giây
    }

    @computed get total() {
      return this.price * this.amount;
    }
}

const item = new OrderLine(5); // Autorun: Total is 5
item.amount = 2; // Autorun: Total is 10
item.amount = 3; // Autorun: Total is 15, Total exceeded 10, disposing autorun.
item.amount = 4; // Không log gì nữa vì đã dispose
```
Hàm được truyền vào `autorun` sẽ chạy ngay lập tức lần đầu tiên khi được định nghĩa, và sau đó là mỗi khi các `observable` được truy cập bên trong nó thay đổi.

### 4. `when`

`when` chạy một hàm (side effect) *một lần duy nhất* khi một điều kiện (dựa trên `observable`) trở thành `true`. Nó rất hữu ích cho các tác vụ cần chạy khi một state cụ thể đạt đến một trạng thái nào đó. Giống như `autorun`, nó trả về một hàm `disposer`.

```javascript
import { observable, computed, when } from "mobx";

class OrderLine {
    @observable price = 0;
    @observable amount = 1;

    constructor(price) {
      this.price = price;
      console.log("Setting up 'when'...");
      // when nhận 2 hàm: hàm điều kiện và hàm effect
      // Hàm effect chỉ chạy ONCE khi hàm điều kiện trả về true lần đầu tiên
      when(
        () => this.total > 10, // Điều kiện
        () => { // Hàm effect
          console.log(`Condition met: total (${this.total}) > 10. Setting total to 0.`);
          // Lưu ý: Việc thay đổi state trong hàm effect của when/autorun/reaction
          // thường nên được bọc trong action nếu nằm trong class/store.
          this.price = 0;
          this.amount = 0;
        }
      );
    }

    @computed get total() {
      console.log("Calculating total...");
      return this.price * this.amount;
    }
}

const item = new OrderLine(5); // Setting up 'when'..., Calculating total...
item.amount = 2; // Calculating total... -> 10 (when condition is false)
item.amount = 3; // Calculating total... -> 15 (when condition is true!)
                 // Condition met: total (15) > 10. Setting total to 0.
                 // Calculating total... -> 0 (Because price/amount were set to 0 by the effect)

console.log(item.total); // Calculating total... -> 0
```
`when` chỉ kiểm tra điều kiện và chạy effect *một lần* khi điều kiện chuyển từ false sang true.

### 5. `reaction`

`reaction` là một phiên bản "nâng cao" hơn của `autorun`. Nó nhận vào hai hàm:
1.  **Data Function:** Chạy đầu tiên. Nó theo dõi các `observable` được truy cập và trả về một giá trị. `reaction` chỉ chạy lại khi *kết quả* của hàm này thay đổi.
2.  **Effect Function:** Chạy khi kết quả của Data Function thay đổi. Nó nhận kết quả mới và cũ của Data Function làm tham số.

Điều này cho phép kiểm soát chặt chẽ hơn việc khi nào side effect nên chạy so với `autorun`.

```javascript
import { observable, computed, reaction } from "mobx";

class TodoList {
    @observable todos = [
      { title: "Make coffee", done: true },
      { title: "Find biscuit", done: false },
    ];

    constructor() {
      console.log("Setting up reactions...");
      // Reaction 1 (Example of a common mistake): Only reacts when the *length* of the todos array changes.
      reaction(
        () => this.todos.length, // Data function: returns the length
        (length, oldLength, reaction) => { // Effect function
            console.log(`Reaction 1 (Length changed): Array length is now ${length}. Todos: ${this.todos.map(todo => todo.title).join(", ")}`);
        }
      );

      // Reaction 2 (Better approach): Reacts when the *mapped titles* change.
      reaction(
        () => this.todos.map(todo => todo.title), // Data function: returns an array of titles
        (titles, oldTitles, reaction) => { // Effect function
            console.log(`Reaction 2 (Titles changed): New titles are [${titles.join(", ")}]. Old titles were [${oldTitles?.join(", ")}].`);
        },
        { equals: (a, b) => JSON.stringify(a) === JSON.stringify(b) } // Optional: Custom comparison for array equality
      );
    }

    // Phương thức thêm todo (ví dụ: bọc trong action)
    // @action
    addTodo(title: string) {
        this.todos.push({ title, done: false });
    }

    // Phương thức sửa title (ví dụ: bọc trong action)
    // @action
    editTodoTitle(index: number, newTitle: string) {
        if (this.todos[index]) {
            this.todos[index].title = newTitle;
        }
    }
}

const list = new TodoList(); // Setting up reactions... (No output yet from reactions because data functions didn't run/change their result)

// Thêm một todo
list.addTodo("explain reactions");
// Output:
// Reaction 1 (Length changed): Array length is now 3. Todos: Make coffee, Find biscuit, explain reactions
// Reaction 2 (Titles changed): New titles are [Make coffee, Find biscuit, explain reactions]. Old titles were [Make coffee, Find biscuit].

// Sửa title của todo đầu tiên
list.editTodoTitle(0, "Make tea");
// Output:
// Reaction 2 (Titles changed): New titles are [Make tea, Find biscuit, explain reactions]. Old titles were [Make coffee, Find biscuit, explain reactions].
// (Reaction 1 không chạy vì độ dài mảng không đổi)

```
Reaction 1 chỉ chạy khi độ dài mảng `todos` thay đổi. Reaction 2 chạy khi mảng các title được tạo ra từ `todos.map(todo => todo.title)` thay đổi (tức là khi có todo được thêm/xóa hoặc title của một todo nào đó đổi).

## Kết Luận

MobX cung cấp một cách tiếp cận trực quan và hiệu quả để quản lý state trong ứng dụng React. Với cú pháp quen thuộc (đặc biệt với các developer React), cơ chế `observable` và `observer` mạnh mẽ, cùng với các hàm tiện ích như `computed`, `autorun`, `when`, và `reaction`, bạn có thể xây dựng các ứng dụng phức tạp với hiệu năng tốt và code dễ bảo trì hơn.

So với Redux, MobX thường yêu cầu ít boilerplate hơn và cho phép mutation trực tiếp, điều này có thể làm cho việc học và sử dụng ban đầu trở nên dễ dàng hơn. Mặc dù hệ sinh thái có thể không lớn bằng React hoặc Redux, MobX vẫn là một lựa chọn mạnh mẽ đáng cân nhắc cho các dự án React của bạn.

Hãy thử sử dụng MobX trong dự án tiếp theo của bạn để trải nghiệm sự khác biệt trong quản lý state!
