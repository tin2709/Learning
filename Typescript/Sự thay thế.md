

# 1. Vấn đề: Sự lặp lại và rủi ro khi tạo Discriminated Union thủ công

Trước tiên, hãy xem cách chúng ta thường tạo một Discriminated Union theo cách truyền thống. Giả sử chúng ta muốn định nghĩa các loại sự kiện (Event) trong một ứng dụng:

```typescript
// --- CÁCH THỦ CÔNG ---

// 1. Phải định nghĩa từng interface/type riêng lẻ
interface LoginEvent {
  type: 'LOGIN'; // Thuộc tính phân biệt (discriminant)
  payload: {
    userId: string;
  };
}

interface LogoutEvent {
  type: 'LOGOUT'; // Lặp lại 'type'
  payload: {
    reason: string;
  };
}

interface ClickEvent {
  type: 'CLICK'; // Lặp lại 'type'
  payload: {
    x: number;
    y: number;
  };
}

// 2. Phải kết hợp chúng lại thành một union
type AppEvent = LoginEvent | LogoutEvent | ClickEvent;

// 3. Khi sử dụng
function handleEvent(event: AppEvent) {
  switch (event.type) {
    case 'LOGIN':
      // TypeScript hiểu ở đây event là LoginEvent
      console.log('User logged in:', event.payload.userId);
      break;
    case 'LOGOUT':
      // TypeScript hiểu ở đây event là LogoutEvent
      console.log('User logged out:', event.payload.reason);
      break;
    // ...
  }
}
```

**Nhược điểm của cách làm này:**

1.  **Lặp lại code (Duplication):** Chúng ta phải gõ `type: 'TÊN_TYPE'` ở mọi interface.
2.  **Dễ mắc lỗi (Error-prone):** Rất dễ gõ sai chuỗi discriminant. Ví dụ, nếu bạn gõ `type: 'LOG_IN'` thay vì `type: 'LOGIN'`, TypeScript sẽ không báo lỗi ngay lập tức, nhưng logic trong `switch...case` sẽ không bao giờ chạy đúng.
3.  **Khó bảo trì:** Khi thêm một sự kiện mới, bạn phải tạo một interface mới và thêm nó vào `AppEvent` union.

---

### 2. Giải pháp: Utility Type `ToDiscoUnion`

`ToDiscoUnion` (viết tắt của "To Discriminated Union") giải quyết tất cả các vấn đề trên bằng cách tự động hóa quá trình này.

#### Định nghĩa của `ToDiscoUnion`

Đây là một phiên bản phổ biến của utility type này. Ta sẽ phân tích nó ngay sau đây.

```typescript
type ToDiscoUnion<
  // T là một object có các key là string, và value là object (payload)
  T extends Record<string, object>,
  // D là tên của thuộc tính phân biệt (mặc định là 'type')
  D extends string = 'type'
> = {
  // 1. Lặp qua từng key (K) trong T (ví dụ: 'LOGIN', 'LOGOUT')
  [K in keyof T]: {
    // 2. Tạo một object mới:
    //    - Thêm thuộc tính phân biệt với key là D và value là K
    //    - Kết hợp (&) nó với payload gốc (T[K])
    [P in D]: K;
  } & T[K];
// 3. Lấy tất cả các giá trị của object vừa tạo ra để biến chúng thành một union
}[keyof T];
```

### 3. Giải thích chi tiết từng phần

Hãy chia nhỏ công thức trên để hiểu rõ hơn. Ta sẽ dùng ví dụ `AppEvent` để minh họa.

Đầu tiên, chúng ta định nghĩa "payload" cho mỗi sự kiện trong một object duy nhất. Đây là **nguồn sự thật duy nhất (single source of truth)**.

```typescript
const eventPayloads = {
  LOGIN: {
    payload: { userId: string }
  },
  LOGOUT: {
    payload: { reason: string }
  },
  CLICK: {
    payload: { x: number, y: number }
  }
} as const; // `as const` rất quan trọng để TypeScript giữ lại các key dạng chuỗi literal
```

Bây giờ, hãy xem `ToDiscoUnion<typeof eventPayloads>` hoạt động như thế nào.

#### **Bước 1: `[K in keyof T]: ...` (Mapped Type)**

TypeScript sẽ lặp qua tất cả các `key` của `eventPayloads`. `K` sẽ lần lượt là `'LOGIN'`, `'LOGOUT'`, `'CLICK'`.

Nó sẽ tạo ra một kiểu đối tượng (object type) mới có cấu trúc như sau:

```typescript
// Kết quả tạm thời của Mapped Type (chưa phải là union)
{
  LOGIN: { type: 'LOGIN' } & { payload: { userId: string } },
  LOGOUT: { type: 'LOGOUT' } & { payload: { reason: string } },
  CLICK: { type: 'CLICK' } & { payload: { x: number, y: number } }
}
```

*   `[P in D]: K;` tạo ra `{ type: 'LOGIN' }`, `{ type: 'LOGOUT' }`, v.v. (ở đây `D` mặc định là `'type'`).
*   `T[K]` lấy ra kiểu dữ liệu của payload tương ứng, ví dụ `typeof eventPayloads['LOGIN']` là `{ payload: { userId: string } }`.
*   `&` (Intersection Type) kết hợp hai phần này lại.

Sau khi kết hợp, kiểu đối tượng tạm thời sẽ trông như thế này:

```typescript
{
  LOGIN: {
    type: 'LOGIN';
    payload: { userId: string };
  },
  LOGOUT: {
    type: 'LOGOUT';
    payload: { reason: string };
  },
  CLICK: {
    type: 'CLICK';
    payload: { x: number, y: number };
  }
}
```

#### **Bước 2: `[keyof T]` (Indexed Access Type / Lookup Type)**

Đây là bước quyết định để biến object type ở trên thành một **union**.

`keyof T` trong trường hợp này là `'LOGIN' | 'LOGOUT' | 'CLICK'`.

Khi bạn áp dụng `[keyof T]` vào một object type, nó sẽ lấy ra một **union của tất cả các kiểu giá trị (value types)** trong object đó.

Nói cách khác, nó tương đương với:

`KiểuCủaLogin | KiểuCủaLogout | KiểuCủaClick`

**Kết quả cuối cùng:**

```typescript
type AppEvent =
  | {
      type: 'LOGIN';
      payload: { userId: string };
    }
  | {
      type: 'LOGOUT';
      payload: { reason: string };
    }
  | {
      type: 'CLICK';
      payload: { x: number, y: number };
    };
```

Đây chính xác là Discriminated Union mà chúng ta muốn!

---

### 4. Ví dụ cụ thể và hoàn chỉnh

```typescript
// ----------------- ĐỊNH NGHĨA -----------------

// Utility type ToDiscoUnion
type ToDiscoUnion<
  T extends Record<string, object>,
  D extends string = 'type'
> = {
  [K in keyof T]: {
    [P in D]: K;
  } & T[K];
}[keyof T];


// ----------------- SỬ DỤNG -----------------

// 1. Định nghĩa các payload trong một object duy nhất (Single Source of Truth)
const eventPayloads = {
  LOGIN: {
    userId: string;
  },
  LOGOUT: {
    reason: string;
  },
  CLICK: {
    x: number;
    y: number;
  }
} as const; // `as const` để bảo toàn các key là literal types

// 2. Tạo Discriminated Union với tên thuộc tính phân biệt là 'kind'
type AppEvent = ToDiscoUnion<typeof eventPayloads, 'kind'>;

/*
TypeScript sẽ tự suy ra AppEvent là:
type AppEvent =
  | { kind: 'LOGIN'; userId: string; }
  | { kind: 'LOGOUT'; reason: string; }
  | { kind: 'CLICK'; x: number; y: number; };
*/


// 3. Sử dụng union một cách an toàn
function processEvent(event: AppEvent) {
  console.log(`Processing event: ${event.kind}`);

  switch (event.kind) {
    case 'LOGIN':
      // OK! TypeScript biết event.userId tồn tại
      console.log('User ID:', event.userId);
      // Lỗi! Property 'reason' does not exist on type '{ kind: "LOGIN"; userId: string; }'
      // console.log(event.reason);
      break;

    case 'LOGOUT':
      // OK! TypeScript biết event.reason tồn tại
      console.log('Reason for logout:', event.reason);
      break;

    case 'CLICK':
      // OK!
      console.log(`Clicked at coordinates: (${event.x}, ${event.y})`);
      break;
      
    // Trường hợp mặc định để đảm bảo mọi loại sự kiện đều được xử lý
    default:
      const _exhaustiveCheck: never = event;
      return _exhaustiveCheck;
  }
}

// Tạo một sự kiện và sử dụng
const loginAction: AppEvent = { kind: 'LOGIN', userId: 'user-123' };
processEvent(loginAction); // Output: Processing event: LOGIN, User ID: user-123

const clickAction: AppEvent = { kind: 'CLICK', x: 100, y: 200 };
processEvent(clickAction); // Output: Processing event: CLICK, Clicked at coordinates: (100, 200)

// Lỗi biên dịch! 'kind' không thể là 'SIGNUP' vì nó không có trong eventPayloads
// const invalidAction: AppEvent = { kind: 'SIGNUP' };
```

### 5. Lợi ích chính

1.  **DRY (Don't Repeat Yourself):** Bạn chỉ cần định nghĩa các payload. Thuộc tính phân biệt được thêm vào tự động.
2.  **An toàn về kiểu (Type Safety):** Không thể có sự sai khác giữa giá trị của thuộc tính phân biệt (`'LOGIN'`) và key trong object định nghĩa (`LOGIN`). Chúng là một. Điều này loại bỏ hoàn toàn lỗi gõ sai.
3.  **Dễ bảo trì và mở rộng:** Muốn thêm một sự kiện mới? Chỉ cần thêm một cặp key-value vào object `eventPayloads`. `AppEvent` sẽ tự động được cập nhật.
4.  **Rõ ràng và tập trung:** `eventPayloads` trở thành nguồn tài liệu duy nhất và rõ ràng cho tất cả các loại sự kiện trong hệ thống.