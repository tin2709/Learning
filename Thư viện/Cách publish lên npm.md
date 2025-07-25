Để tạo ra một thư viện Node.js/JavaScript/TypeScript có thể được cài đặt thông qua `npm i tên-thư-viện` và sử dụng, như thư viện `toast-noti` này, có một quy trình và các cấu hình chính cần tuân thủ.

Dựa vào các file bạn cung cấp, quy trình và cấu hình để tạo ra thư viện `toast-noti` như sau:

### 1. Cấu trúc dự án và mã nguồn (Source Code)

*   **`src/toast.ts`**: Đây là file mã nguồn chính chứa logic của thư viện (lớp `Toast`, các phương thức `success`, `error`, `alert`, v.v.). Nó được viết bằng TypeScript để tận dụng tính năng kiểm tra kiểu (type-checking) và cung cấp các tệp định nghĩa kiểu (`.d.ts`) cho người dùng TypeScript.
*   **`src/toast.style.css`**: Đây là file CSS chứa tất cả các style cần thiết cho các toast. Việc tách CSS ra thành một file riêng và người dùng phải `import` nó thủ công là một cách tiếp cận phổ biến để cho phép người dùng tùy chỉnh CSS dễ dàng hơn.

### 2. Cấu hình TypeScript (tsconfig.json)

File `tsconfig.json` dùng để cấu hình trình biên dịch TypeScript:

```json
{
    "include": [
        "src/toast.ts" // Chỉ định các file TS cần biên dịch
    ],
    "compilerOptions": {
        "module": "ESNext",      // Module output format (ESM for modern JS)
        "target": "ESNext",      // Target JavaScript version (ESM for modern browsers/Node)
        "outDir": "dist",        // Thư mục đầu ra cho các file đã biên dịch (JS và .d.ts)
        "declaration": true,     // Rất quan trọng: Tạo file định nghĩa kiểu (.d.ts)
        "esModuleInterop": true, // Hỗ trợ tương tác giữa CommonJS và ES Modules
        "removeComments": true   // Xóa bỏ comment trong file đầu ra
    }
}
```

*   **`"outDir": "dist"`**: Điều này đảm bảo rằng khi TypeScript biên dịch `src/toast.ts`, các file `toast.js` và `toast.d.ts` sẽ được đặt vào thư mục `dist/`.
*   **`"declaration": true`**: Đây là chìa khóa để tạo ra file `toast.d.ts`, file này chứa các định nghĩa kiểu TypeScript, cho phép các dự án khác sử dụng thư viện này với TypeScript mà không gặp lỗi về kiểu.

### 3. Cấu hình Package (package.json)

File `package.json` là trung tâm của mọi gói npm. Nó định nghĩa metadata, script, dependencies, và quan trọng nhất là cách gói này được sử dụng khi cài đặt.

```json
{
  "name": "toast-noti", // Tên gói (package name), dùng khi npm i tên-gói
  "version": "1.0.4",
  "description": "A super simple toast noti library with pure Javascript",
  "main": "dist/toast.js", // Điểm vào chính cho các môi trường CommonJS (Node.js) và các bundler cũ hơn
  "types": "dist/toast.d.ts", // Điểm vào cho các file định nghĩa kiểu TypeScript
  "files": [ // Rất quan trọng: Liệt kê các file/thư mục sẽ được bao gồm khi publish lên npm
    "./dist/toast.d.ts",
    "./dist/toast.js",
    "./dist/toast.style.css" // Đảm bảo file CSS cũng được đưa vào gói
  ],
  "repository": { ... },
  "scripts": {
    "build": "tsc && esbuild src/* --minify --outdir=dist" // Script build
  },
  "keywords": [ ... ],
  "author": "HOAI AN",
  "license": "ISC",
  "bugs": { ... },
  "homepage": "https://github.com/HOAIAN2/toast-noti#readme",
  "devDependencies": { // Các công cụ phát triển, không được cài đặt khi người dùng cài đặt thư viện
    "esbuild": "^0.19.4",
    "typescript": "^5.2.2"
  }
}
```

*   **`"name": "toast-noti"`**: Đây là tên mà người dùng sẽ sử dụng trong lệnh `npm i toast-noti`.
*   **`"main": "dist/toast.js"`**: Khi một dự án khác `import Toast from 'toast-noti'`, Node.js (hoặc bundler như Webpack/Rollup) sẽ tìm đến file được chỉ định ở đây. Nó trỏ đến phiên bản JavaScript đã được biên dịch và tối ưu trong thư mục `dist`.
*   **`"types": "dist/toast.d.ts"`**: Khi một dự án TypeScript `import Toast from 'toast-noti'`, TypeScript sẽ sử dụng file này để có được thông tin về kiểu dữ liệu của thư viện.
*   **`"files"`**: **Đây là phần cực kỳ quan trọng.** Nó chỉ định những file hoặc thư mục nào từ dự án của bạn sẽ được đóng gói và tải lên registry của npm khi bạn chạy `npm publish`. Trong trường hợp này, chỉ các file đã build (`.js`, `.d.ts`, `.css`) trong thư mục `dist` mới được đưa vào. Điều này giúp gói của bạn nhỏ gọn và chỉ chứa những gì cần thiết cho người dùng cuối. Các file nguồn (`src/`), `node_modules/`, `tsconfig.json`, v.v. sẽ không được đưa vào.
*   **`"scripts": { "build": "tsc && esbuild src/* --minify --outdir=dist" }`**:
    *   `tsc`: Chạy TypeScript compiler. Nó sẽ đọc `src/toast.ts` (dựa trên `include` trong `tsconfig.json`) và biên dịch nó thành `dist/toast.js` và `dist/toast.d.ts` (dựa trên `outDir` và `declaration` trong `tsconfig.json`).
    *   `esbuild src/* --minify --outdir=dist`: `esbuild` là một công cụ bundler/minifier rất nhanh.
        *   `src/*`: Chỉ định các file đầu vào (có thể là `src/toast.ts`, `src/toast.style.css` và các file khác trong `src`).
        *   `--minify`: Tối ưu hóa kích thước file bằng cách nén mã.
        *   `--outdir=dist`: Chỉ định thư mục đầu ra là `dist`. Điều này đảm bảo rằng các file JS đã được tối ưu và file CSS (nếu `esbuild` xử lý nó trực tiếp hoặc copy nó) sẽ nằm trong thư mục `dist`.

### 4. Quy trình tạo thư viện để phân phối qua NPM

1.  **Phát triển mã nguồn:** Viết logic thư viện trong `src/toast.ts` và styling trong `src/toast.style.css`.
2.  **Cấu hình Build:** Thiết lập `tsconfig.json` và `package.json` như đã mô tả ở trên, đặc biệt chú ý đến `outDir`, `declaration`, `main`, `types`, và `files`.
3.  **Chạy lệnh Build:**
    Chạy lệnh `npm run build` trong terminal.
    *   Lệnh `tsc` sẽ biên dịch `src/toast.ts` thành `dist/toast.js` và `dist/toast.d.ts`.
    *   Lệnh `esbuild` sẽ lấy các file trong `src` (bao gồm `dist/toast.js` và `src/toast.style.css` - có thể `esbuild` copy CSS trực tiếp hoặc bundler JS và CSS). Kết quả là bạn sẽ có các file sẵn sàng để phân phối trong thư mục `dist`: `toast.js`, `toast.d.ts`, và `toast.style.css`.
4.  **Kiểm tra trước khi Publish (Tùy chọn nhưng khuyến khích):**
    Bạn có thể dùng `npm pack` để tạo một file `.tgz` cục bộ, file này mô phỏng gói sẽ được publish. Sau đó, bạn có thể cài đặt file `.tgz` này vào một dự án thử nghiệm khác (`npm i ../đường/dẫn/đến/gói.tgz`) để đảm bảo mọi thứ hoạt động đúng như mong đợi.
5.  **Publish lên NPM:**
    Sau khi bạn chắc chắn mọi thứ đã sẵn sàng, chạy lệnh `npm publish`.
    *   Lệnh này sẽ lấy các file được liệt kê trong thuộc tính `"files"` của `package.json` (nghĩa là các file trong thư mục `dist`), nén chúng lại thành một gói và tải lên registry của npm với tên được định nghĩa trong `"name"` của `package.json`.
    *   Lần đầu publish, bạn cần đăng nhập npm (`npm login`).
    *   Sau mỗi lần thay đổi, bạn nên tăng số `version` trong `package.json` trước khi publish lại.

### 5. Cách người dùng sử dụng sau khi cài đặt

Khi người dùng chạy `npm i toast-noti`:

*   npm sẽ tải gói đã publish từ registry và đặt nó vào `node_modules/toast-noti/`.
*   Trong thư mục đó, họ sẽ chỉ thấy các file đã build (`dist/toast.js`, `dist/toast.d.ts`, `dist/toast.style.css`) cùng với `package.json` của thư viện.
*   Khi người dùng viết:
    *   `import Toast from 'toast-noti'`: Trình biên dịch TypeScript hoặc bundler (như Webpack, Vite, Rollup) sẽ sử dụng thông tin từ `"main"` (`dist/toast.js`) và `"types"` (`dist/toast.d.ts`) trong `package.json` của thư viện để cung cấp lớp `Toast`.
    *   `import 'toast-noti/dist/toast.style.css'`: Trình bundler sẽ tìm đến file CSS này trong gói `node_modules/toast-noti/dist/` và đưa nó vào bundle của ứng dụng của người dùng.

Tóm lại, quá trình tạo thư viện để phân phối qua npm bao gồm việc viết mã nguồn, cấu hình TypeScript để biên dịch và tạo kiểu, cấu hình `package.json` để định nghĩa điểm vào và các file cần đóng gói, và sử dụng các script build để chuẩn bị các file cuối cùng cho việc publish.