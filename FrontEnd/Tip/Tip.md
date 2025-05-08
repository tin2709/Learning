
# 1  Chia sẻ Component Giữa các Website: Từ Đơn giản đến Hiệu quả

Việc chuyển giao component giữa các website là một nhu cầu phổ biến trong quá trình phát triển ứng dụng web. Bài viết này sẽ hướng dẫn bạn một số phương pháp chia sẻ component, từ cơ bản đến thực tiễn, giúp tối ưu hóa quá trình phát triển.

*(Lưu ý: Nội dung gốc có liên kết đến bài viết giải thích "Component là gì?". Để đơn giản, chúng ta sẽ xem component là một phần giao diện người dùng có thể tái sử dụng, thường bao gồm HTML, CSS và có thể cả JavaScript.)*

Khi tạo ứng dụng web, việc cần chuyển component giữa các website là điều không hiếm gặp. Thông thường, đó là một số loại nút chung, các khối như footer, header, v.v.

## Ví dụ: Component Nút Cơ bản

Ví dụ, chúng ta có thể lấy component nút, thứ mà chúng ta sẽ chuyển giữa các website. Nó sẽ trông như thế này:

```html
<button class="button">Click Me</button>
```

Cùng với các kiểu dáng (styles) của nó:

```css
.button {
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 12px 24px;
  text-align: center;
  text-decoration: none;
  font-size: 16px;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s, transform 0.2s;
}

.button:hover {
  background-color: #45a049;
}

.button:active {
  transform: scale(0.95);
}
```

Bây giờ, hãy lấy hai website mà bạn cần làm cho component này trở nên dùng chung. Giả sử đó là `example1.com` và `example2.com`. Chúng có thể được lưu trữ trên các máy chủ (hosting) khác nhau. Một website được triển khai từ GitHub Pages và website còn lại từ một hosting cục bộ nào đó.

Câu hỏi chính đặt ra - làm thế nào để chia sẻ component này một cách hiệu quả giữa hai môi trường khác nhau như vậy?

## Các Phương pháp Chia sẻ Component

Tôi sẽ mô tả một số phương pháp thường được sử dụng cho việc này. Từ phương pháp thông thường nhất đến phương pháp thực tiễn hơn.

### Phương pháp 1: Xuất ra File Script

Phương pháp này giả định rằng logic tạo ra component (HTML và CSS) sẽ nằm trong một hàm JavaScript bên trong một file `.js`. File này sau đó có thể được nhúng trực tiếp vào bất kỳ trang HTML nào bằng thẻ `<script>`. Hàm này sẽ chịu trách nhiệm tạo ra các phần tử HTML cần thiết (như `<button>`) và có thể cả phần tử `<style>`.

Hãy tạo file `buttonModule.js`:

```javascript
// buttonModule.js
(function (global) {
  // Định nghĩa hàm createButton
  function createButton() {
    // Tạo phần tử <style> và thêm CSS styles
    const style = document.createElement('style');
    style.textContent = `
      .button {
        background-color: #4caf50;
        color: white;
        border: none;
        padding: 12px 24px;
        text-align: center;
        text-decoration: none;
        font-size: 16px;
        border-radius: 5px;
        cursor: pointer;
        transition: background-color 0.3s, transform 0.2s;
      }
      .button:hover {
        background-color: #45a049;
      }
      .button:active {
        transform: scale(0.95);
      }
    `;

    // Tạo phần tử button
    const button = document.createElement('button');
    button.className = 'button';
    button.textContent = 'Click Me';

    // Trả về các phần tử (style và button)
    return { style, button };
  }

  // Công khai hàm ra phạm vi toàn cục
  global.buttonModule = {
    createButton,
  };
})(window);
```

Sau đó, trên cả `example1.com` và `example2.com`, bạn sẽ nhúng file script này (giả sử nó được lưu trữ tại một URL công khai nào đó, ví dụ `https://.../buttonModule.js`) và sử dụng hàm đã được công khai:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Button Module</title>
  <!-- Liên kết đến file script dùng chung -->
  <script src="https://.../buttonModule.js"></script>
</head>
<body>
  <div id="wrapper"></div>
  <script>
    // Sử dụng buttonModule để tạo button
    const { style, button } = buttonModule.createButton();
    const wrapper = document.getElementById("wrapper");

    // Thêm style và button vào trang
    // Lưu ý: Việc thêm phần tử <style> như thế này có thể ảnh hưởng đến toàn bộ trang.
    // Cân nhắc các chiến lược CSS khác cho các trường hợp phức tạp.
    wrapper.append(style);
    wrapper.append(button);
  </script>
</body>
</html>
```

**Ưu điểm của Phương pháp 1:**

*   **Dễ dàng thực hiện:** Dễ dàng với thẻ `<script>` tiêu chuẩn trong HTML mà không cần thiết lập phức tạp.
*   **Thiết lập tối thiểu:** Phù hợp với các ứng dụng nhỏ, một trang hoặc các thử nghiệm nhanh.
*   **Tích hợp nhanh:** Có thể được tích hợp liền mạch vào các dự án hiện có vốn đã dựa vào các biến toàn cục.

**Nhược điểm của Phương pháp 1:**

*   **Khó mở rộng:** Nếu có nhiều component, sẽ có rất nhiều thẻ `<script>`, làm cho phương pháp này chỉ phù hợp cho một số ít component.
*   **Làm ô nhiễm phạm vi toàn cục:** Thêm các biến hoặc đối tượng vào phạm vi toàn cục, làm tăng nguy cơ xung đột tên.
*   **Quản lý phụ thuộc khó khăn:** Khó quản lý các phụ thuộc giữa các script.
*   **Khó bảo trì:** Khiến dự án khó mở rộng hoặc tái cấu trúc hơn.
*   **Phụ thuộc vào thứ tự tải:** Các script phụ thuộc vào thứ tự tải chính xác, điều này phải được quản lý thủ công.
*   **CSS Isolation kém:** Việc thêm thẻ `<style>` trực tiếp ảnh hưởng đến toàn bộ trang, có thể gây ra xung đột style không mong muốn.

### Phương pháp 2: Sử dụng Thư viện Bên thứ Ba (ví dụ HMPL) thông qua API

Phương pháp này tận dụng một thư viện templating hướng máy chủ (server-oriented) như HMPL. Thay vì nhúng logic tạo component vào một script phía client, bạn sẽ lưu trữ mã HTML và CSS thô của component (hoặc kết quả được server render) trên một máy chủ. Client sau đó sẽ lấy nội dung này thông qua một cơ chế nhẹ nhàng do thư viện cung cấp.

Với phương pháp này, chúng ta sẽ lưu trữ component nút cơ bản dưới dạng một file HTML đơn giản (`button.html`):

```html
<!-- button.html -->
<button class="button">Click Me</button>
<style>
  .button {
    background-color: #4caf50;
    color: white;
    border: none;
    padding: 12px 24px;
    text-align: center;
    text-decoration: none;
    font-size: 16px;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s, transform 0.2s;
  }

  .button:hover {
    background-color: #45a049;
  }

  .button:active {
    transform: scale(0.95);
  }
</style>
```

Tiếp theo, thiết lập một endpoint đơn giản trên máy chủ để trả về file này. Sử dụng Node.js với Express:

```javascript
// server/buttonController.js
const express = require("express");
const path = require("path");
const expressRouter = express.Router();

const buttonController = (req, res) => {
  // Trả về file button.html khi truy cập endpoint này
  res.sendFile(path.join(__dirname, "../button.html"));
};

// Định nghĩa tuyến đường API (ví dụ: /api/getButton)
expressRouter.use("/getButton", buttonController);

module.exports = expressRouter; // Xuất router

// server/app.js (Ví dụ điểm vào server đơn giản)
const express = require("express");
const cors = require("cors"); // Cần thiết nếu client khác domain/port
const routes = require("./routes/buttonController"); // Import controller của bạn

const PORT = process.env.PORT || 8000;
const app = express();

// Thêm middleware
app.use(cors()); // Cho phép các yêu cầu cross-origin
// Có thể thêm middleware phục vụ file tĩnh nếu cần (tùy chọn cho ví dụ này)
// app.use(express.static(path.join(__dirname, "public")));

// Sử dụng các tuyến đường button dưới namespace /api
app.use("/api", routes);

app.listen(PORT, () => {
  console.log(`Máy chủ đang chạy trên cổng ${PORT}`);
});
```

Trên phía client (trong `example1.com` và `example2.com`), bạn sẽ nhúng thư viện HMPL (qua script hoặc import) và sử dụng cú pháp template của nó để fetch component:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Ví dụ HMPL Button</title>
    <!-- Bao gồm JSON5 và DOMPurify là các dependency của HMPL -->
    <script src="https://unpkg.com/json5/dist/index.min.js"></script>
    <script src="https://unpkg.com/dompurify/dist/purify.min.js"></script>
    <!-- Bao gồm HMPL -->
    <script src="https://unpkg.com/hmpl-js/dist/hmpl.min.js"></script>
  </head>
  <body>
    <div id="wrapper"></div>
    <script>
      // Compile một template HMPL sẽ gửi yêu cầu fetch component từ API
      const templateFn = hmpl.compile(
        `<div id="wrapper">{{#request src="https://your-api-server.com/api/getButton"}}{{/request}}</div>`
      );

      // Render template, điều này sẽ kích hoạt yêu cầu fetch
      const btnWrapper = templateFn().response;

      // Thêm phần tử kết quả vào trang (sẽ chứa button và styles đã fetch)
      document.body.append(btnWrapper);

      // Bạn có thể dễ dàng tái sử dụng hàm template để tạo nhiều instance
      // const anotherBtnWrapper = templateFn().response;
      // document.body.append(anotherBtnWrapper);
    </script>
  </body>
</html>
```

**Ưu điểm của Phương pháp 2 (Sử dụng HMPL/API):**

*   **Tái sử dụng Component:** Các component được fetch và render động, giúp việc tái sử dụng trở nên đơn giản và an toàn.
*   **Khả năng mở rộng:** Hoạt động hiệu quả cho các ứng dụng có nhiều component.
*   **Tính năng phong phú:** HMPL cung cấp các tính năng tích hợp sẵn như chỉ báo tải (loading indicators), xử lý lỗi yêu cầu, kích hoạt dựa trên sự kiện (`after`), tự động tạo body form (`autoBody`), caching (`memo`), và đặc biệt quan trọng là vệ sinh HTML (`sanitize` thông qua DOMPurify) để bảo vệ khỏi XSS.
*   **Tính linh hoạt:** Cho phép render component động phía server dựa trên ngữ cảnh yêu cầu.
*   **Giảm kích thước JavaScript ở client:** Logic cốt lõi của component (HTML/CSS) nằm trên server, làm giảm kích thước bundle phía client so với việc nhúng các framework component JS.

**Nhược điểm của Phương pháp 2:**

*   **Yêu cầu thiết lập máy chủ:** Cần có một endpoint trên máy chủ để lưu trữ và phục vụ HTML của component.
*   **Các Dependency bổ sung:** Cần bao gồm HMPL (và các dependency của nó như JSON5 và DOMPurify) ở phía client.
*   **Không phải là SSR:** Phương pháp này chủ yếu là kết xuất phía client được kích hoạt bởi fetch, nghĩa là component sẽ không hiển thị trong mã HTML ban đầu mà các robot hoặc trình duyệt không bật JavaScript nhận được (khác với Server-Side Rendering truyền thống).

## Kết luận

Phương pháp tốt nhất để chia sẻ component phụ thuộc vào nhu cầu cụ thể của bạn:

*   **Phương pháp File Script** đơn giản và yêu cầu thiết lập tối thiểu, phù hợp cho việc chia sẻ một số lượng rất nhỏ các component đơn giản hoặc cho các thử nghiệm nhanh. Tuy nhiên, nó nhanh chóng trở nên khó quản lý và dễ gây xung đột khi số lượng hoặc độ phức tạp của các component dùng chung tăng lên.
*   **Phương pháp HMPL/API** yêu cầu thiết lập ban đầu phức tạp hơn (một endpoint trên server và bao gồm HMPL), nhưng mang lại những lợi thế đáng kể về khả năng tái sử dụng, mở rộng và các tính năng tích hợp sẵn để xử lý việc tải nội dung động và bảo mật. Đây là một phương pháp mạnh mẽ cho các ứng dụng muốn giữ cho JavaScript phía client nhẹ nhàng trong khi tận dụng khả năng kết xuất phía máy chủ cho các component UI.

Cuối cùng, việc lựa chọn phương pháp phù hợp đòi hỏi phải cân nhắc giữa sự dễ dàng triển khai với khả năng mở rộng, bảo trì và các tính năng cần thiết.

---

Để biết thêm thông tin về HMPL, hãy xem kho lưu trữ GitHub [hmpl-language/hmpl](https://github.com/hmpl-language/hmpl).
