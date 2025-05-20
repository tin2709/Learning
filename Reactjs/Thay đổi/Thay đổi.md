
# 1 Tại Sao Tôi Quyết Định Chuyển Từ React/Next.js Sang Svelte (Một Góc Nhìn Với Svelte 5)
Sponsor by https://medium.com/@urboifox/from-react-to-svelte-a-simpler-faster-way-to-build-for-the-web-b937f6861f39

Trong một thời gian dài, React và Next.js là những công cụ chính của tôi để xây dựng ứng dụng web. Sự phổ biến rộng rãi, cộng đồng hỗ trợ mạnh mẽ và hệ thống component linh hoạt đã khiến chúng trở thành lựa chọn đáng tin cậy. Tuy nhiên, theo thời gian, tôi bắt đầu nhận thấy một số nhược điểm. Tôi phải xử lý quá nhiều code lặp đi lặp lại, cách quản lý dữ liệu phức tạp, và cảm giác như đang dành nhiều thời gian để "làm việc xung quanh" framework hơn là xây dựng dự án thực sự. Tôi muốn một công cụ giúp việc phát triển dễ dàng và nhanh chóng hơn mà không làm mất đi khả năng tạo ra các ứng dụng chất lượng cao, hiệu quả.

Đó là lúc tôi biết đến Svelte. Tôi lần đầu tìm hiểu về nó qua các video của Fireship và quyết định khám phá sâu hơn. Những gì tôi khám phá được là một framework giải quyết được nhiều vấn đề tôi gặp phải với React, đồng thời khiến việc phát triển trở nên thú vị trở lại. README này phản ánh trải nghiệm của tôi với Svelte phiên bản 5, phiên bản mới nhất tại thời điểm viết, dù lần đầu tôi tiếp cận là với một phiên bản cũ hơn. Hãy bắt đầu bằng cách xem xét những thách thức tôi đã gặp phải với React trước khi giải thích tại sao Svelte lại trở thành lựa chọn ưu tiên của tôi.

## Những Thách Thức Tôi Gặp Phải Với React

### Hiệu Năng Chậm Trong Giao Diện Phức Tạp

Trong một dự án, tôi đã làm việc trên một hệ thống quản lý nhân viên có một trang quyền hạn (permissions page). Trang này có khoảng 50 quyền, mỗi quyền có ba ô checkbox — chỉnh sửa, xóa và xem — tổng cộng 150 ô checkbox. Mỗi ô checkbox được liên kết với một trạng thái (state), và bất cứ khi nào người dùng click vào một ô, trang mất khoảng hai giây để cập nhật. Ngoài ra còn có ô tìm kiếm để lọc quyền, nhưng việc gõ vào đó gây ra độ trễ với mỗi ký tự, khiến trải nghiệm người dùng rất khó chịu.

Vấn đề nằm ở cách React cập nhật màn hình. Khi dữ liệu thay đổi, React so sánh phiên bản ảo của trang (virtual DOM) với trang thực tế và vẽ lại bất cứ thứ gì khác biệt. Và với nhiều phần tương tác, điều này có thể làm chậm mọi thứ.

Mặt khác, Svelte chỉ cập nhật chính xác những phần thay đổi, nhờ hệ thống dựa trên tín hiệu (signals - trong Svelte 5). Điều này làm cho nó nhanh hơn đáng kể, ngay cả khi xử lý hàng trăm input như trong trang quyền hạn của tôi.

### Kích Thước File Lớn Ảnh Hưởng Đến Thời Gian Tải Trang

Một vấn đề khác với React là kích thước của các file được tạo ra. Ngay cả một ứng dụng React cơ bản cũng đi kèm với khoảng 50–60 KB mã, chủ yếu là do virtual DOM và các tính năng tích hợp khác.

Svelte áp dụng một cách tiếp cận khác — nó biến code của bạn thành JavaScript thuần túy, hiệu quả trong quá trình build. Đối với một ứng dụng đơn giản, điều này có thể tạo ra các file nhỏ chỉ khoảng 3 KB. Không có gánh nặng phụ của virtual DOM, ứng dụng Svelte tải nhanh và chạy mượt mà.

### Thiết Lập và Quy Tắc Quá Rườm Rà

Cách làm của React rất mạnh mẽ, nhưng thường đòi hỏi nhiều bước bổ sung. Quản lý dữ liệu, thiết lập điều hướng, hoặc lấy thông tin từ server có nghĩa là phải viết nhiều code hơn tôi muốn — hoặc liên tục kiểm tra tài liệu để làm được việc. Ngay cả với Next.js, vốn đã đơn giản hóa một số việc này, tôi vẫn cảm thấy như đang làm việc cho công cụ thay vì công cụ làm việc cho mình.

> React khiến bạn cảm thấy như nó là con đường duy nhất để xây dựng phần mềm.

## Điều Gì Khiến Svelte Khác Biệt và Tốt Hơn

Dưới đây là lý do tại sao Svelte đã trở thành framework ưu tiên của tôi cho việc phát triển web.

### Quản Lý Dữ Liệu Dễ Dàng

Trong React, theo dõi dữ liệu có thể trở nên phức tạp. Bạn có thể dùng `useState` cho những việc nhỏ, nhưng đối với các kịch bản lớn hơn, bạn có thể cần đến Context API hoặc một thư thư viện riêng biệt như Redux hay Zustand. Mỗi lựa chọn lại thêm nhiều bước và quyết định. Svelte làm điều này đơn giản hơn nhiều với khả năng biến state thành global. Ví dụ, để tạo một phần dữ liệu được chia sẻ trong Svelte 5, bạn chỉ cần viết:

```typescript
// File: lib/store.svelte.ts
export const myStore = $state({ items: [] });
```

Bạn có thể sử dụng store này ở bất cứ đâu trong ứng dụng, và nó tự động cập nhật khi dữ liệu thay đổi. Giống như có một phiên bản Redux nhẹ được tích hợp sẵn trong Svelte — không cần thiết lập thêm. Điều này giảm bớt sự phức tạp và làm cho mọi thứ dễ đoán hơn.

Bạn không cần làm việc với các thư viện khác, hay phải thích ứng với thư viện quản lý state của từng dự án. Mọi thứ đều là Svelte.

### Xử Lý Input Đơn Giản Hơn

Form trong React yêu cầu bạn viết code để theo dõi mọi thay đổi trong trường input. Ví dụ, để tạo một controlled input, bạn sẽ cần thứ gì đó như thế này:

```jsx
const [name, setName] = useState("");
// ...
<input type="text" value={name} onChange={(e) => setName(e.target.value)} />
```

Điều này nhanh chóng trở nên lặp đi lặp lại, đặc biệt với nhiều trường. Svelte làm điều này dễ dàng hơn với khả năng binding dữ liệu hai chiều (two-way data binding) tích hợp sẵn:

```svelte
<script>
  let name = $state("");
</script>

<input type="text" bind:value={name} />
```

Dòng `bind:value` kết nối trực tiếp input với biến `name`. Khi bạn gõ, `name` cập nhật, và nếu `name` thay đổi ở nơi khác, input cũng cập nhật theo. Đây là cách xử lý form sạch sẽ và nhanh hơn.

Bạn cũng có `bind:checked` và `bind:group`, và thậm chí có thể biến một prop thành bindable bằng cách sử dụng rune `$bindable`. Nhưng tôi sẽ không giải thích sâu hơn vì bài viết sẽ quá dài. Bạn đã hiểu ý rồi chứ.

### Xử Lý Dữ Liệu Lồng Nhau Mà Không Cần Thêm Công Sức

Trong React, cập nhật dữ liệu có cấu trúc lồng nhau — như một object bên trong một object — có thể phức tạp. Bạn phải tạo một bản sao của dữ liệu trước tiên, như thế này:

```javascript
const [data, setData] = useState({ location: { country: { city: "New York" } } });
const newData = { ...data };
newData.location.country.city = "Boston";
setData(newData);
```

Điều này không dễ đoán, vì nếu bạn thay đổi state trực tiếp thì sẽ không hoạt động, đòi hỏi thêm công sức và có thể dẫn đến sai sót. Svelte xử lý điều này tự động với hệ thống reactivity của nó. Bạn chỉ cần viết:

```svelte
<script>
  let data = $state({ location: { country: { city: "New York" } } });
</script>

<button onclick={() => data.location.country.city = "Boston"}>Change City</button>
```

Svelte cập nhật trang hiệu quả mà bạn không cần phải quản lý các bản sao. Điều này làm cho việc làm việc với dữ liệu phức tạp cảm thấy tự nhiên và đơn giản.

### Hiệu Ứng Chuyển Động và Hình Ảnh Tích Hợp Sẵn

Thêm hoạt ảnh (animations) trong React thường có nghĩa là phải mang thêm các công cụ bên ngoài như Framer Motion hoặc GSAP, làm tăng thêm code cần tải về và cần học. Svelte có các hiệu ứng chuyển động được tích hợp sẵn, vì vậy bạn có thể làm những việc như thế này:

```svelte
<script>
  import { fade } from 'svelte/transition';
  let show = $state(true);
</script>

{#if show}
  <div transition:fade>This fades in and out</div>
{/if}

<button onclick={() => show = !show}>Toggle</button>
```

Điều này tạo ra hiệu ứng fade mượt mà mà gần như không tốn công sức. Svelte cung cấp các tùy chọn khác như trượt (sliding) hoặc thay đổi kích thước (scaling), và tất cả đều hoạt động với hệ thống cập nhật trang của framework. Đây là cách nhẹ nhàng để làm cho ứng dụng của bạn trông trau chuốt hơn mà không cần công cụ bổ sung.

### Trải Nghiệm Tốt Hơn Cho Nhà Phát Triển

Svelte mang lại cảm giác được thiết kế để giúp đỡ bạn, chứ không phải kìm hãm bạn. Bạn có thể xây dựng bất kỳ ứng dụng web nào với bất kỳ framework nào. Nhưng trải nghiệm của nhà phát triển đối với tôi là chìa khóa. Ngay cả khi hiệu năng của Svelte rất lớn, nhưng hầu hết những lợi ích mà bạn sẽ thấy mọi người nói đến lại là trải nghiệm của nhà phát triển. Đối với tôi, đó là điều bạn sẽ không biết cảm giác thế nào cho đến khi tự mình thử.

## Tại Sao Svelte Là Lựa Chọn Thắng Thế Với Tôi

Tôi không nghĩ có thể gói gọn tất cả các tính năng của Svelte so với React vào một bài viết này, nhưng tôi đoán bạn đã hiểu được vấn đề rồi. Svelte giải quyết các vấn đề hiệu năng tôi thấy ở React, giữ kích thước file nhỏ để tải nhanh hơn, và đơn giản hóa các tác vụ như quản lý dữ liệu và thêm hiệu ứng. Ngoài những lợi ích kỹ thuật, nó còn mang lại niềm vui khi tạo ra các ứng dụng web. Tôi dành ít thời gian hơn để "đối đầu" với framework và nhiều thời gian hơn để biến ý tưởng của mình thành hiện thực.

Tôi khuyến khích bạn thử Svelte. Đó là một công cụ tuyệt vời mang lại cảm giác dễ tiếp cận và hiệu quả — một sự kết hợp khó có thể đánh bại trong thế giới phát triển web hiện nay.
```