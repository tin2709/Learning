Dưới đây là bản phân tích chi tiết về dự án **Mitosis** (của Builder.io) dựa trên cấu trúc thư mục và mã nguồn bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology)

Mitosis không phải là một framework UI chạy ở runtime (như React hay Vue), mà là một **Source-to-Source Compiler (Transpiler)**.

*   **Babel & AST (Abstract Syntax Tree):** Đây là "xương sống" của Mitosis. Hệ thống sử dụng Babel để phân tích cú pháp JSX (định dạng `.lite.tsx`) thành một cây đại diện trừu tượng (AST), sau đó biến đổi cây này thành một định dạng trung gian (Intermediate Representation - IR) là JSON.
*   **TypeScript & ts-morph:** Dự án sử dụng TypeScript để đảm bảo tính chặt chẽ. `ts-morph` được sử dụng trong các bước phân tích sâu về kiểu dữ liệu (như nhận diện các thuộc tính optional hoặc Signal).
*   **Prettier:** Được tích hợp trực tiếp vào quá trình biên dịch để đảm bảo mã nguồn đầu ra (React, Vue, Svelte...) luôn đẹp và đúng chuẩn format của framework đó.
*   **Nx (Monorepo Tool):** Quản lý toàn bộ dự án dưới dạng monorepo, giúp tối ưu hóa việc xây dựng (build), kiểm thử (test) và quản lý phụ thuộc giữa các gói `core`, `cli`, `docs`, và `e2e`.

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Mitosis dựa trên nguyên lý **"Single Source of Truth" (Một nguồn sự thật duy nhất)**.

*   **Định dạng trung gian (JSON IR):** Thay vì chuyển đổi trực tiếp từ React sang Vue, Mitosis chuyển đổi từ JSX sang một cấu trúc JSON chung (`MitosisComponent`). JSON này mô tả đầy đủ: State, Props, Hooks, Imports và cấu trúc DOM. Điều này cho phép mở rộng thêm các framework mới chỉ bằng cách viết thêm một "Generator" cho JSON đó.
*   **Static JSX Subset:** Mitosis giới hạn JSX ở một tập hợp con mang tính "tĩnh" (static). Bạn không thể dùng các logic JavaScript phức tạp tùy ý bên trong hàm render. Thay vào đó, bạn phải dùng các component điều khiển luồng như `<Show>` và `<For>`. Tư duy này giúp code dễ dàng được ánh xạ sang template-based frameworks (như Angular/Svelte) lẫn vDOM-based frameworks (như React).
*   **Write Once, Compile Everywhere:** Kiến trúc tách biệt hoàn toàn giữa **Parser** (người đọc code) và **Generator** (người viết code).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **AST Transformation (Biến đổi cây cú pháp):** Đây là kỹ thuật khó nhất. Mitosis tìm các định danh (identifiers) như `state.foo` và thay đổi chúng tùy theo target.
    *   *Ví dụ:* Sang React thành `foo`, sang Vue thành `this.foo` hoặc `foo.value` (composition API).
*   **Plugin Architecture:** Cả `core` và `cli` đều hỗ trợ plugin. Điều này cho phép người dùng can thiệp vào các giai đoạn: `preJson`, `postJson`, `preCode`, `postCode`. Ví dụ: Plugin `compileAwayBuilderComponents` dùng để loại bỏ các component đặc thù của Builder.io khi xuất ra mã nguồn sạch.
*   **Functional Programming (FP):** Sử dụng thư viện `fp-ts` và các kỹ thuật như `pipe`, `flow` để xử lý chuỗi biến đổi dữ liệu một cách minh bạch và ít lỗi.
*   **Reactivity Mapping:** Kỹ thuật ánh xạ mô hình phản ứng. Mitosis chuyển đổi các hook `useStore`, `useState` sang Reactive đặc thù của từng framework (Signals của Qwik/Angular, Proxy của Vue, Hook của React).

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng đi của một file code qua Mitosis có thể tóm tắt như sau:

1.  **Giai đoạn Input (JSX/Svelte):** Hệ thống nhận file đầu vào (thường là `.lite.tsx`).
2.  **Giai đoạn Parsing (Phân tích):**
    *   `packages/core/src/parsers/jsx/jsx.ts` sử dụng Babel để đọc file.
    *   Bóc tách các phần: `imports`, `state` (từ `useStore`), `hooks` (từ `onMount`, `useEffect`), và phần JSX template.
    *   Kết quả là một đối tượng **Mitosis JSON**.
3.  **Giai đoạn Transformation (Biến đổi nội bộ):**
    *   Chạy các plugin tiền xử lý (pre-json).
    *   Xử lý CSS-in-JS: Thu thập các thuộc tính `css={{...}}` và chuyển thành đối tượng style chung.
    *   Ánh xạ context và các tham chiếu `ref`.
4.  **Giai đoạn Code Generation (Tạo mã):**
    *   JSON được đưa vào các Generator tương ứng (ví dụ: `generators/vue/vue.ts`).
    *   Generator sẽ duyệt qua cây JSON và lắp ghép thành chuỗi string mã nguồn của framework đích.
    *   Kỹ thuật "Identify Replacement" được thực hiện để sửa các lệnh truy cập biến cho phù hợp.
5.  **Giai đoạn Output (Định dạng & Xuất bản):**
    *   Mã nguồn (string) được chạy qua Prettier để format.
    *   Các plugin hậu xử lý (post-code) chạy lần cuối.
    *   CLI ghi file ra thư mục `output/`.

### Tổng kết
Mitosis là một kỳ công về kỹ thuật thao tác mã (code manipulation). Nó tận dụng tối đa sức mạnh của hệ sinh thái JavaScript Tooling (Babel, TypeScript) để giải quyết bài toán phân mảnh framework, biến JSX thành một "ngôn ngữ lập trình UI chung".