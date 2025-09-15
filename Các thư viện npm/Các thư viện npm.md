# 1 Class-variance-authority
`class-variance-authority` (thường được viết tắt là **CVA**) là một thư viện nhỏ gọn nhưng cực kỳ mạnh mẽ, giúp bạn xây dựng các UI component có thể tái sử dụng với các **biến thể (variants)** một cách dễ dàng và có tổ chức, đặc biệt khi làm việc với các framework utility-class như **Tailwind CSS**.

Hãy tưởng tượng CVA như một "nhà máy" tạo ra các chuỗi class CSS dựa trên các thuộc tính (props) bạn truyền vào component.

---

### Tác dụng của nó là gì? (Vấn đề CVA giải quyết)

Để hiểu rõ tác dụng của CVA, hãy xem xét vấn đề mà nó giải quyết.

#### **Kịch bản "Trước khi có CVA"**

Giả sử bạn muốn tạo một component `Button` trong React với Tailwind CSS. Button này có nhiều biến thể:
*   **Mục đích (intent):** `primary`, `secondary`, `danger`
*   **Kích thước (size):** `small`, `medium`, `large`

Cách làm thông thường (không có CVA) sẽ trông như thế này:

```jsx
// Button.jsx (chưa dùng CVA)
import React from 'react';

const Button = ({ intent, size, children }) => {
  // Logic để nối chuỗi class rất phức tạp và khó đọc
  const baseClasses = "font-semibold border rounded shadow";
  
  const intentClasses = intent === 'primary' 
    ? "bg-blue-500 text-white border-blue-600 hover:bg-blue-700"
    : intent === 'secondary'
    ? "bg-gray-200 text-gray-800 border-gray-300 hover:bg-gray-300"
    : "bg-red-500 text-white border-red-600 hover:bg-red-700";

  const sizeClasses = size === 'small'
    ? "py-1 px-2 text-sm"
    : size === 'medium'
    ? "py-2 px-4 text-base"
    : "py-3 px-6 text-lg";

  const className = `${baseClasses} ${intentClasses} ${sizeClasses}`;

  return <button className={className}>{children}</button>;
};
```

**Vấn đề của cách làm trên:**
*   **Khó đọc, khó bảo trì:** Logic dùng `if/else` hoặc toán tử ba ngôi (`? :`) để nối chuỗi `className` trở nên rối rắm khi có nhiều biến thể.
*   **Dễ xảy ra lỗi:** Dễ quên dấu cách, hoặc mắc lỗi logic khi thêm biến thể mới.
*   **Khó mở rộng:** Thêm một biến thể mới (ví dụ: `outline`) đòi hỏi phải sửa đổi nhiều dòng code.
*   **Logic bị trộn lẫn:** Logic về class bị viết trực tiếp bên trong component, làm component trở nên cồng kềnh.

---

### **Giải pháp của Class-Variance-Authority (CVA)**

CVA cho phép bạn định nghĩa tất cả các biến thể này một cách khai báo (declarative) và tách biệt hoàn toàn khỏi logic của component.

#### **Kịch bản "Sau khi có CVA"**

Cùng component `Button` ở trên, nhưng giờ chúng ta sẽ dùng CVA:

**1. Cài đặt thư viện:**
```bash
npm install class-variance-authority
```

**2. Định nghĩa các biến thể:**

```javascript
// buttonVariants.js
import { cva } from 'class-variance-authority';

export const buttonVariants = cva(
  // 1. Lớp cơ sở (Base classes): Áp dụng cho TẤT CẢ các biến thể
  'font-semibold border rounded shadow transition-colors duration-150',
  {
    // 2. Các biến thể (Variants)
    variants: {
      intent: {
        primary: 'bg-blue-500 text-white border-blue-600 hover:bg-blue-700',
        secondary: 'bg-gray-200 text-gray-800 border-gray-300 hover:bg-gray-300',
        danger: 'bg-red-500 text-white border-red-600 hover:bg-red-700',
      },
      size: {
        small: 'py-1 px-2 text-sm',
        medium: 'py-2 px-4 text-base',
        large: 'py-3 px-6 text-lg',
      },
    },
    
    // 3. Các biến thể kết hợp (Compound Variants): Áp dụng khi nhiều điều kiện cùng đúng
    compoundVariants: [
      {
        intent: 'primary',
        size: 'large',
        className: 'uppercase', // Ví dụ: button primary và large thì viết hoa
      },
    ],
    
    // 4. Các giá trị mặc định (Default Variants)
    defaultVariants: {
      intent: 'primary',
      size: 'medium',
    },
  }
);
```

**3. Sử dụng trong Component React:**

Bây giờ, component `Button` của bạn trở nên cực kỳ gọn gàng.

```jsx
// Button.jsx (đã dùng CVA)
import React from 'react';
import { buttonVariants } from './buttonVariants';
import { cva, type VariantProps } from 'class-variance-authority' // Dùng cho TypeScript

// Lấy ra kiểu (type) của các props từ CVA (rất hữu ích với TypeScript)
export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement>,
  VariantProps<typeof buttonVariants> {}

const Button = ({ className, intent, size, children, ...props }: ButtonProps) => {
  return (
    // Chỉ cần gọi hàm buttonVariants và truyền props vào!
    <button className={buttonVariants({ intent, size, className })} {...props}>
      {children}
    </button>
  );
};
```

**Cách hoạt động:**
Hàm `buttonVariants({ intent: 'danger', size: 'small' })` sẽ tự động trả về chuỗi class chính xác: `"font-semibold border rounded shadow ... bg-red-500 text-white ... py-1 px-2 text-sm"`.

---

### Mối quan hệ với `clsx` và `tailwind-merge`

CVA thường được sử dụng cùng với hai thư viện khác để trở nên hoàn hảo:

1.  **`clsx` (hoặc `classnames`):**
    *   **Tác dụng:** Giúp nối các chuỗi `className` lại với nhau một cách thông minh, loại bỏ các giá trị `null`, `undefined` hoặc `false`.
    *   CVA đã tích hợp sẵn một phiên bản tương tự `clsx`, nên bạn không cần cài đặt nó riêng cho CVA.

2.  **`tailwind-merge`:**
    *   **Tác dụng:** Đây là thư viện cực kỳ quan trọng khi dùng Tailwind. Nó giúp giải quyết xung đột giữa các utility class.
    *   **Ví dụ:** Nếu bạn có chuỗi class là `p-2 p-4`, `tailwind-merge` sẽ tự động hiểu rằng `p-4` sẽ ghi đè `p-2` và kết quả cuối cùng chỉ là `p-4`. Tương tự, `bg-red-500 bg-blue-500` sẽ cho ra `bg-blue-500`.
    *   Khi bạn cho phép người dùng truyền `className` tùy chỉnh vào component (như ví dụ `Button` ở trên), `tailwind-merge` sẽ hợp nhất các class từ CVA và class tùy chỉnh một cách chính xác.

> **Thực tế:** Các dự án lớn như **shadcn/ui** thường tạo một hàm tiện ích `cn` để kết hợp `clsx` và `tailwind-merge`, sau đó truyền nó vào CVA hoặc dùng để bao bọc kết quả của CVA.

---

### Tóm tắt lợi ích của Class-Variance-Authority (CVA)

*   **Tổ chức code sạch sẽ:** Tách biệt hoàn toàn logic về style ra khỏi component.
*   **Dễ bảo trì và mở rộng:** Thêm một variant mới chỉ cần thêm một dòng trong file cấu hình CVA.
*   **Khả năng tái sử dụng cao:** Định nghĩa các biến thể một lần và sử dụng ở nhiều nơi.
*   **An toàn về kiểu (Type-Safety):** Tích hợp hoàn hảo với TypeScript, giúp tự động gợi ý các props (`intent`, `size`) và giá trị của chúng.
*   **Tính dự đoán được:** Code trở nên rõ ràng, dễ dàng biết được component sẽ trông như thế nào chỉ bằng cách nhìn vào các props được truyền vào.


# 2  File-Saver là thư viện gì?

**File-Saver** (hay `FileSaver.js`) là một thư viện JavaScript nhỏ gọn, giúp bạn giải quyết một vấn đề rất phổ biến trong phát triển web: **lưu các tệp tin (files) xuống máy tính của người dùng trực tiếp từ trình duyệt (client-side)**.

Nói một cách đơn giản, nó cung cấp một hàm `saveAs()` đa năng để bạn có thể kích hoạt hành động "Tải về" cho các dữ liệu được tạo ra ngay trong ứng dụng web của bạn.

### Tác dụng chính và tại sao nó hữu ích?

Vấn đề cốt lõi mà `file-saver` giải quyết là các trình duyệt không có một API gốc (native) đơn giản và nhất quán để lập trình viên có thể yêu cầu "lưu một đoạn dữ liệu bất kỳ thành một file". `File-Saver` lấp đầy khoảng trống này.

**1. Lưu các tệp được tạo ra từ phía Client:**
Đây là công dụng quan trọng nhất. Hãy tưởng tượng ứng dụng của bạn cho phép người dùng:
*   Soạn thảo một văn bản trong một trình editor online.
*   Vẽ một bức tranh trên thẻ `<canvas>`.
*   Tạo ra một file CSV hoặc JSON từ dữ liệu trên trang.

Với `file-saver`, bạn có thể dễ dàng thêm một nút "Tải về" để người dùng lưu lại tác phẩm hoặc dữ liệu của họ thành file `.txt`, `.png`, `.csv`, v.v.

**2. Giải quyết sự không tương thích giữa các trình duyệt:**
Việc lưu file hoạt động khác nhau trên các trình duyệt (Chrome, Firefox, Safari, Edge...). `File-Saver` xử lý các khác biệt này và cung cấp một API duy nhất, giúp code của bạn chạy ổn định ở mọi nơi.

**3. Phân biệt rõ với việc tải file từ Server:**
Tài liệu của `file-saver` có một lưu ý quan trọng:
> Nếu file bạn muốn người dùng tải về **đã có sẵn trên server**, bạn nên ưu tiên sử dụng header `Content-Disposition: attachment` trong response từ server. Cách này có độ tương thích cao hơn.

`File-Saver` chỉ thực sự tỏa sáng khi **dữ liệu được tạo ra hoặc xử lý ngay tại trình duyệt**.

---

### Cách hoạt động và cú pháp cơ bản

Thư viện cung cấp một hàm chính là `saveAs()`.

**Cú pháp:**
```javascript
import { saveAs } from 'file-saver';

saveAs(data, filename, options);
```

*   `data`: Dữ liệu bạn muốn lưu. Có thể là:
    *   **Blob:** Một đối tượng chứa dữ liệu thô, thường được dùng để lưu văn bản, JSON, CSV...
    *   **File:** Một đối tượng `File` (ví dụ từ một input upload file).
    *   **URL:** Một đường dẫn (URL) tới một tài nguyên (ví dụ: một bức ảnh).
*   `filename` (tùy chọn): Tên file bạn muốn đặt, ví dụ `"mydocument.txt"`.
*   `options` (tùy chọn): Các tùy chọn bổ sung, ví dụ `{ autoBom: true }` để xử lý mã hóa ký tự Unicode.

#### Ví dụ cụ thể:

**1. Lưu một file văn bản:**
```javascript
import { saveAs } from 'file-saver';

// Tạo một Blob chứa dữ liệu text
var blob = new Blob(["Chào bạn, đây là nội dung file!"], {type: "text/plain;charset=utf-8"});

// Gọi hàm saveAs để tải file
saveAs(blob, "chao-ban.txt");
```

**2. Lưu một ảnh từ thẻ `<canvas>`:**
Đây là một trường hợp sử dụng rất phổ biến.
```javascript
// Giả sử bạn có một canvas với id="my-canvas"
var canvas = document.getElementById("my-canvas");

// Chuyển nội dung canvas thành một Blob
canvas.toBlob(function(blob) {
    // Lưu Blob đó thành file ảnh
    saveAs(blob, "buc-tranh-dep.png");
});
```
*Lưu ý:* Một số trình duyệt cũ không hỗ trợ `canvas.toBlob()`. Bạn có thể cần một thư viện bổ trợ khác là `canvas-toBlob.js`.

**3. Lưu một file từ URL:**
```javascript
import { saveAs } from 'file-saver';

// Thư viện sẽ tự động fetch dữ liệu từ URL và kích hoạt tải về
saveAs("https://via.placeholder.com/150", "image.jpg");
```

---

### Những điểm cần lưu ý (Hạn chế)

*   **Safari:** Đôi khi, thay vì tải file về, Safari sẽ mở file đó trong một tab mới. Người dùng có thể phải tự tay bấm `⌘+S` để lưu.
*   **iOS (iPhone/iPad):** Vì các giới hạn của Apple, việc tải file phải được kích hoạt bởi một hành động của người dùng (như click chuột, chạm màn hình). Nó cũng thường mở file trong một cửa sổ mới thay vì tải trực tiếp.
*   **File lớn:** Đối với các file cực lớn (vài trăm MB đến GB), `file-saver` có thể không phải là giải pháp tối ưu vì nó cần tải toàn bộ dữ liệu vào RAM trước khi lưu. Trong trường hợp này, tài liệu gợi ý sử dụng thư viện nâng cao hơn là `StreamSaver.js`.

### Cài đặt
Bạn có thể cài đặt thư viện này dễ dàng qua npm:
```bash
# Cài đặt thư viện chính
npm install file-saver --save

# Nếu bạn dùng TypeScript, cài thêm định nghĩa kiểu
npm install @types/file-saver --save-dev
```

### Tóm tắt

**File-Saver là một công cụ thiết yếu cho các ứng dụng web cần chức năng "xuất file" hoặc "tải dữ liệu" được tạo ra bởi người dùng.** Nó cung cấp một API đơn giản, che giấu sự phức tạp của việc tương tác với hệ thống file trên các trình duyệt khác nhau, giúp bạn tập trung vào logic của ứng dụng.



# 3 `minimal-shared` là thư viện gì?

**Tóm tắt:**

`minimal-shared` là một thư viện **phụ trợ/nội bộ**. Chức năng chính của nó là chứa các **React Hooks** và các **hàm tiện ích (utils)** được sử dụng chung bởi hai thư viện giao diện người dùng (UI) khác là **Minimal UI** và **Zone UI**.

Nói cách khác, đây không phải là một thư viện độc lập mà bạn sẽ cài đặt để xây dựng ứng dụng của mình từ đầu. Thay vào đó, nó là một "hộp công cụ" chung để tránh việc viết đi viết lại cùng một đoạn code trong hai dự án có liên quan với nhau.

### Phân tích chi tiết

#### 1. Mục đích cốt lõi: Tái sử dụng Code (Don't Repeat Yourself - DRY)

Đây là lý do tồn tại của thư viện này. Hãy tưởng tượng:
*   `Minimal UI` là một bộ sưu tập các component (nút, form, card...).
*   `Zone UI` cũng là một bộ sưu tập component khác, có thể có phong cách hoặc mục đích khác.

Cả hai thư viện này có thể cần những logic hoặc hàm tiện ích giống hệt nhau. Ví dụ:
*   Một **custom hook** để quản lý trạng thái bật/tắt (toggle).
*   Một **hàm tiện ích (util)** để định dạng ngày tháng.
*   Một **hàm tiện ích** để kết hợp các `className` (giống như `clsx` hay `tailwind-merge`).

Thay vì viết lại các hàm này ở cả hai dự án (`Minimal UI` và `Zone UI`), nhà phát triển đã tách chúng ra một gói riêng là `minimal-shared`.
*   **Lợi ích:** Dễ bảo trì, đảm bảo tính nhất quán, và khi cần sửa lỗi hoặc cải tiến một hàm, họ chỉ cần sửa ở một nơi duy nhất.

#### 2. Thành phần chính của thư viện

Dựa trên mô tả, thư viện này chứa hai loại tài nguyên chính:

*   **Hooks:** Các hàm React Hook tùy chỉnh (custom hooks) giúp đóng gói và tái sử dụng logic có trạng thái. Ví dụ có thể là `useDebounce`, `useLocalStorage`, `useEventListener`...
*   **Utils (Utilities):** Các hàm JavaScript thuần túy, không phụ thuộc vào React, dùng để thực hiện các tác vụ nhỏ, lặp đi lặp lại. Ví dụ: `capitalize()`, `formatNumber()`, `getUniqueId()`...

#### 3. Mối quan hệ với người dùng

*   **Ai là người dùng trực tiếp?** Các nhà phát triển (maintainers) của `Minimal UI` và `Zone UI`.
*   **Ai là người dùng gián tiếp?** Bất kỳ ai cài đặt `Minimal UI` hoặc `Zone UI`. Khi bạn cài một trong hai thư viện này, trình quản lý gói (như npm hoặc yarn) sẽ tự động cài `minimal-shared` như một **phụ thuộc (dependency)**. Bạn có thể không nhận ra sự tồn tại của nó, nhưng nó vẫn đang hoạt động "ngầm" bên trong.

> **Bạn có nên cài đặt `minimal-shared` một cách độc lập không?**
> **Câu trả lời là không**, trừ khi bạn biết chính xác mình cần một hook hoặc một util cụ thể từ nó. Thư viện này được thiết kế để phục vụ cho hệ sinh thái của `Minimal UI` và `Zone UI`.

#### 4. Phân tích các thông tin khác từ Socket

*   **Socket Score (92/100):** Điểm số rất cao. Điều này cho thấy thư viện có chất lượng tốt, được bảo trì tốt, ít lỗ hổng bảo mật và tuân thủ các thực hành tốt nhất về chuỗi cung ứng phần mềm. Đây là một dấu hiệu đáng tin cậy.
*   **Weekly downloads (11K):** Số lượt tải hàng tuần khá tốt. Con số này không phản ánh người dùng trực tiếp cài đặt `minimal-shared`, mà là tổng số lượt cài đặt của các thư viện mẹ (`Minimal UI`, `Zone UI`). Nó cho thấy hệ sinh thái này đang được sử dụng bởi một cộng đồng nhất định.
*   **Maintenance:** "Version published 3 weeks ago" (Phiên bản được xuất bản 3 tuần trước) cho thấy dự án vẫn đang được bảo trì và cập nhật tích cực.

### Kết luận

**`minimal-shared` là một ví dụ điển hình về thực hành kỹ thuật phần mềm tốt.** Nó là một thư viện phụ thuộc, đóng vai trò là nền tảng chung để chia sẻ code giữa các dự án liên quan, giúp cho việc phát triển và bảo trì trở nên hiệu quả và nhất quán hơn. Nó không phải là công cụ dành cho người dùng cuối mà là một phần không thể thiếu trong "hệ sinh thái" của Minimal UI và Zone UI.

# 4 progress-tracker
Sponsor by https://github.com/NigelOToole/progress-tracker

Thư viện tạo progress từng step cho người dùng

# 5 Remotion
# Remotion - Hướng dẫn Bắt đầu Nhanh


## 🚀 Bắt đầu

Remotion cho phép bạn tạo video, hoạt ảnh (animations) và đồ họa chuyển động (motion graphics) bằng cách sử dụng các công nghệ web quen thuộc như React. Hướng dẫn này tập trung vào các bước cài đặt ban đầu.

### Điều kiện tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn đã cài đặt một trong những công cụ sau:

*   **Node.js:** Phiên bản 16 trở lên.
*   **Bun:** Phiên bản 1.0.3 trở lên.

**🐧 Lưu ý dành cho Linux:**
*   Yêu cầu phiên bản **2.35 trở lên của Libc**.
*   Bạn có thể cần cài đặt thêm một số **gói phụ trợ**.
*   **Alpine Linux** và **nixOS** hiện **không được hỗ trợ**.

### Tạo Project Mới (Scaffolding)

Cách dễ nhất để bắt đầu một project Remotion mới là sử dụng công cụ dòng lệnh `create-video`.

1.  **Chạy lệnh tạo project:**

    ```bash
    npx create-video@latest
    # hoặc
    # bun create video@latest
    # pnpm create video@latest
    # yarn create video@latest
    ```

2.  **Chọn mẫu (template):**
    Trình cài đặt sẽ hỏi bạn chọn một mẫu. Đối với dự án đầu tiên của bạn, mẫu **"Hello World"** được khuyến nghị mạnh mẽ vì nó cung cấp một điểm khởi đầu đơn giản.

    Các mẫu thông thường như `Next.js + React Router 7` cũng có sẵn cho các thiết lập phức tạp hơn.

3.  **Khởi động Remotion Studio:**
    Sau khi project đã được tạo xong, điều hướng vào thư mục project mới của bạn và khởi động Remotion Studio:

    ```bash
    cd ten-project-cua-ban # Thay thế bằng tên thư mục project thực tế của bạn
    npm run dev
    # hoặc
    # bun run dev
    # pnpm dev
    # yarn dev
    ```
    Lệnh này sẽ khởi chạy Remotion Studio trong trình duyệt của bạn, cho phép bạn xem trước và phát triển video của mình.

### Cài đặt trong các Project Hiện có

Nếu bạn muốn tích hợp Remotion vào một dự án đã có sẵn, vui lòng tham khảo phần riêng biệt trong tài liệu chính thức:

*   [Cài đặt trong các project hiện có](https://www.remotion.dev/docs/getting-started/installation-in-existing-projects)

---

## ✨ Những gì bạn có thể làm với Remotion

Mặc dù hướng dẫn này tập trung vào việc cài đặt, Remotion cho phép bạn:

*   **Tạo hiệu ứng động (animate) cho các thuộc tính** một cách dễ dàng.
*   **Tái sử dụng các component** để cấu trúc dự án.
*   **Xem trước video của bạn** theo thời gian thực.
*   **Thiết kế các hiệu ứng hình ảnh phức tạp**.
*   **Nhúng video và âm thanh**.
*   Tạo **video có tham số hóa (parameterized videos)**.
*   Thực hiện **kết xuất phía máy chủ (server-side rendering)** và sử dụng **Remotion Lambda** để tạo video quy mô lớn.
*   Tích hợp với các **công cụ AI**.

---

## 📚 Tài nguyên & Hỗ trợ

*   **Tài liệu chính thức:** [https://www.remotion.dev/docs](https://www.remotion.dev/docs)
*   **Cộng đồng Discord:** [Hỏi trên Discord](https://www.remotion.dev/discord)
*   **Kho lưu trữ GitHub:** [https://github.com/remotion-dev/remotion](https://github.com/remotion-dev/remotion)

