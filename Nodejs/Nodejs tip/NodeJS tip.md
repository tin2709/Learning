
# 1  Writing Clean, Secure Node.js APIs – A Checklist You’ll Actually Use ✅

This README provides an essential checklist for building clean, secure, and maintainable APIs with Node.js, based on a blog post discussing best practices for API development.

When building APIs with Node.js, you are creating critical infrastructure that applications rely on. Ensuring these APIs are clean, secure, and robust from the start is paramount. This checklist outlines actionable steps you can take to achieve this.

## The Essential Checklist

Here is your checklist for writing clean, secure Node.js APIs:

### ✅ 1. Structure Your Project Like a Pro

A well-organized project structure is key to maintainability and scalability. Adopt a consistent structure early on:

*   `controllers/` — Contain the core business logic and handle incoming requests.
*   `routes/` — Define API endpoints and map them to controllers.
*   `services/` — Handle data operations, interactions with external APIs, or complex business logic abstracted from controllers.
*   `middlewares/` — Implement logic like authentication, validation, logging, and error handling.
*   `models/` — Define database schemas and data access logic.
*   `utils/` — Store reusable helper functions.

**Pro Tip:** Choose a structure that is predictable and easy for any developer to navigate, even if it feels "boring."

### ✅ 2. Validate All Incoming Data

Never trust data coming from the client or any external source. Validation is your first line of defense.

*   Use robust validation libraries like [Joi](https://github.com/sideway/joi), [Zod](https://github.com/colinhacks/zod), or [express-validator](https://express-validator.github.io/docs/).
*   Validate data from all possible sources: request headers, query parameters, and request bodies.
*   Define clear validation rules for required fields, data types, formats (like email), minimum/maximum lengths, and other constraints.

```javascript
const Joi = require('joi');

const userSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
});

// Example usage (within a middleware or controller):
// const { error, value } = userSchema.validate(req.body);
// if (error) { /* handle validation error */ }
```

### ✅ 3. Always Handle Errors Properly

Graceful error handling is crucial for both user experience and debugging. Avoid unhandled exceptions that crash your application.

*   Implement centralized error handling using middleware.
*   Never expose internal stack traces or sensitive information in production error responses sent to clients.
*   Clearly differentiate between client errors (using `4xx` HTTP status codes like 400 Bad Request, 401 Unauthorized, 404 Not Found) and server errors (using `5xx` codes like 500 Internal Server Error).
*   Log errors on the server-side for monitoring and debugging.

```javascript
// Example centralized error handling middleware
app.use((err, req, res, next) => {
  console.error(err); // Log the error internally

  // Determine status code (default to 500)
  const statusCode = err.status || 500;

  // Send a generic error message to the client for non-operational errors
  const message = err.isOperational ? err.message : 'Something went wrong.';

  res.status(statusCode).json({ message });
});
```

### ✅ 4. Secure Your API Like a Bank Vault

API security is non-negotiable. Implement multiple layers of security measures.

*   Use security middleware like [Helmet.js](https://helmetjs.github.io/) to set secure HTTP headers (e.g., preventing clickjacking, XSS attacks).
*   Implement Rate Limiting (e.g., using [express-rate-limit](https://github.com/express-rate-limit/express-rate-limit)) to prevent brute-force attacks and abuse.
*   Configure [CORS (Cross-Origin Resource Sharing)](https://expressjs.com/en/resources/middleware/cors.html) strictly to allow requests only from trusted origins.
*   Implement robust Authentication using established standards like JWT ([jsonwebtoken](https://github.com/auth0/node-jsonwebtoken)) or OAuth2, rather than building your own custom token systems.
*   Implement Input Sanitization (e.g., using libraries like [xss-clean](https://github.com/hurtak/xss-clean)) to protect against Cross-Site Scripting (XSS) and other injection attacks.

```javascript
const helmet = require('helmet');
const express = require('express');
const app = express();

// Use Helmet middleware early in your stack
app.use(helmet());

// Other middleware and routes...
```

### ✅ 5. Use Environment Variables (The Right Way)

Never hardcode secrets, configuration values, or credentials directly in your codebase.

*   Use environment variables for sensitive information (API keys, database passwords, secrets, port numbers, etc.).
*   Utilize libraries like [dotenv](https://github.com/motdotla/dotenv) for local development to load variables from a `.env` file, or a more sophisticated configuration library like [convict](https://github.com/mozilla/node-convict).
*   Ensure your `.env` file (or equivalent) is **NEVER** committed to version control (add it to your `.gitignore`).

**Pro Tip:** Treat your sensitive environment variables like your toothbrush — don’t share them and change them often.

### ✅ 6. Version Your API

Plan for future changes and maintain compatibility with existing clients.

*   Include versioning in your API routes, typically as a prefix (e.g., `/api/v1/users`, `/api/v2/products`).
*   Plan for backward compatibility when introducing new versions.
*   Deprecate old API versions gracefully, providing clear warnings (e.g., via response headers) and a sunset timeline for removal.

### ✅ 7. Write Tests (Yes, You Really Should)

Automated tests are essential for catching bugs and ensuring reliability, especially as your API grows.

*   You don't need 100% test coverage immediately; start somewhere.
*   Begin with:
    *   **Unit tests:** Test individual, isolated functions or modules (e.g., service functions, utility helpers) using frameworks like [Jest](https://jestjs.io/) or [Mocha](https://mochajs.org/).
    *   **Integration tests:** Test the interaction between multiple components, particularly your API endpoints, using libraries like [Supertest](https://github.com/visionmedia/supertest) with a test runner like Jest or Mocha.
*   **Bonus:** Well-written tests serve as living documentation, illustrating how different parts of your API are expected to behave.

### ✅ 8. Log Like a Detective

Effective logging is critical for monitoring, debugging, and understanding API usage in production.

*   Avoid relying solely on `console.log()`. Use structured logging libraries like [Winston](https://github.com/winstonjs/winston) or [Pino](https://github.com/pinojs/pino).
*   Log important events and errors, such as:
    *   Incoming requests (with key details like method, path, status code).
    *   Successful or failed logins/authentications.
    *   Database query errors.
    *   External API call failures.
    *   Unhandled exceptions (captured by your error handling).
*   **Crucially, avoid logging sensitive user data** (passwords, credit card numbers, PII) directly in your logs.

```javascript
const winston = require('winston');

// Example Winston logger configuration
const logger = winston.createLogger({
  level: 'info', // Log level (e.g., error, warn, info, verbose, debug, silly)
  format: winston.format.json(), // Use JSON format for structured logging
  transports: [
    new winston.transports.Console(),
    // Add more transports for file logging, sending to external services, etc.
    // new winston.transports.File({ filename: 'error.log', level: 'error' }),
    // new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Example usage:
// logger.info('User logged in', { userId: user.id, ip: req.ip });
// logger.error('Database query failed', { query: sql, error: err.message });
```

### ✅ 9. Keep Dependencies Up-to-Date

Outdated third-party packages are a common source of security vulnerabilities.

*   Regularly audit your project's dependencies using tools like `npm audit` or integrate with services like [Snyk](https://snyk.io/) or GitHub's [Dependabot](https://github.com/features/security) for automated security vulnerability checks and updates.
*   Stay informed about and upgrade to the latest Long-Term Support (LTS) versions of Node.js.

### ✅ 10. Document Your API

Clear and up-to-date documentation is essential for developers consuming your API.

*   Use tools like [Swagger/OpenAPI](https://swagger.io/) (with libraries like [swagger-ui-express](https://github.com/scottie1984/swagger-ui-express) or [express-openapi-validator](https://github.com/cdimascio/express-openapi-validator)) or [Postman Collections](https://www.postman.com/collections/) to describe your endpoints, parameters, responses, and authentication.
*   Include examples for requests and responses, explanations of error codes, and details on authentication flows.
*   **Crucially, keep your documentation synchronized with your API code.** Update docs whenever endpoints change, not months later.

## Final Thoughts

Building clean, secure Node.js APIs is more than just following a list; it's about building reliable systems and respecting the developers who will use and maintain your code, as well as the users who trust you with their data.

Bookmark this checklist, audit your existing APIs against these points, and strive to ship code you can be proud of. Clean, secure APIs are no longer an option; they are the expected standard.

---

*Based on the blog post "Writing Clean, Secure Node.js APIs – A Checklist You’ll Actually Use ✅" by Mehul Gupta, published March 31, 2025.*

# 2 Sử Dụng Redis Cache Thông Minh Trong Node.js

Redis là một key-value store siêu nhanh, thường được sử dụng làm cache layer cho các ứng dụng backend. Tuy nhiên, nếu bạn chỉ biết các lệnh cơ bản như `.set()` và `.get()`, thì bạn mới chỉ khai thác được một phần nhỏ sức mạnh của Redis.

Trong tài liệu này, chúng ta sẽ khám phá cách sử dụng Redis cache một cách thông minh hơn trong ứng dụng Node.js để tối ưu hiệu năng, tránh cache sai dữ liệu và đảm bảo cache hoạt động hiệu quả khi traffic cao.

## Tại sao nên dùng Redis làm Cache?

*   **Tăng tốc độ phản hồi:** Dữ liệu trả về từ Redis có thể nhanh gấp 10–100 lần so với gọi trực tiếp từ Database (DB).
*   **Giảm tải cho Database:** Những truy vấn lặp lại nhiều lần không cần phải gọi lại DB, giúp giảm áp lực lên Database.
*   **Giữ trạng thái (state) cho nhiều instance:** Phù hợp cho các hệ thống có nhiều instance chạy sau Load Balancer, giúp chia sẻ thông tin hoặc trạng thái giữa các instance (ví dụ: session user).

## Các Kỹ Thuật Sử Dụng Redis Cache Thông Minh

### 1. Cache theo Key Động (Dynamic Key)

*   **Ý tưởng:** Thay vì sử dụng các key tĩnh chung chung (ví dụ: 'user'), hãy tạo key động dựa trên các yếu tố đặc trưng của dữ liệu cần cache như ID của đối tượng, tham số truy vấn (query), hoặc session ID.
*   **Mục đích:** Đảm bảo dữ liệu cache là dành riêng cho từng trường hợp cụ thể, tránh tình trạng cache sai dữ liệu hoặc sử dụng dữ liệu cache của người dùng/truy vấn khác.
*   **Ví dụ:**
    ```javascript
    const userId = req.params.id; // Giả định lấy ID từ request
    const key = `user:${userId}`;
    const cached = await redis.get(key);

    if (cached) {
      console.log('Cache hit!');
      return res.json(JSON.parse(cached));
    }

    console.log('Cache miss, fetching from DB...');
    const user = await db.getUserById(userId);

    if (user) {
      // Cache dữ liệu với thời gian sống 10 phút (600 giây)
      await redis.set(key, JSON.stringify(user), 'EX', 60 * 10);
      // Hoặc cách khác: await redis.setex(key, 60 * 10, JSON.stringify(user));
    }

    return res.json(user);
    ```

### 2. Chỉ Cache Chọn Lọc (Cache Selectively)

*   **Ý tưởng:** Không phải mọi response hoặc mọi loại dữ liệu đều nên được cache. Hãy chỉ cache những dữ liệu phù hợp dựa trên tần suất truy cập, mức độ thay đổi, hoặc giá trị kinh doanh.
*   **Nên tránh cache:**
    *   Dữ liệu thay đổi liên tục, thời gian thực (ví dụ: điểm số trực tiếp trận đấu, giá cổ phiếu thay đổi từng giây). Cache sẽ nhanh chóng lỗi thời.
    *   Dữ liệu chưa đầy đủ hoặc đang trong quá trình xử lý.
*   **Nên cân nhắc cache:** Dữ liệu ít thay đổi, dữ liệu của người dùng có đặc điểm đặc biệt (như tài khoản Premium cần truy xuất nhanh).
*   **Ví dụ:**
    ```javascript
    // ... lấy dữ liệu user từ DB ...
    if (user && user.isPremium) {
      // Chỉ cache dữ liệu user premium vì họ thường truy cập nhiều và cần tốc độ
      await redis.set(`user:${user.id}`, JSON.stringify(user), 'EX', 600); // cache 10 phút
    }
    // ... trả về response ...
    ```

### 3. Cache theo Chuỗi Truy vấn (Query String)

*   **Ý tưởng:** Đối với các API có nhiều biến thể dựa trên tham số truyền lên qua query string (ví dụ: phân trang, lọc, sắp xếp), key cache nên phản ánh chính xác các tham số đó.
*   **Ví dụ:** API lấy danh sách sản phẩm có thể có các tham số `category`, `page`, `sort`.
*   **Cách tạo key:**
    ```javascript
    // Giả sử query: ?category=electronics&page=2&sort=price_asc
    // Cần tạo key dựa trên các tham số này
    const category = req.query.category || 'all'; // Giá trị mặc định nếu không có
    const page = req.query.page || '1';
    const sort = req.query.sort || 'default';

    // Tạo key kết hợp các tham số theo định dạng nhất quán
    const queryKey = `products:category:${category}:page:${page}:sort:${sort}`;
    // Hoặc đơn giản hơn nếu tham số ít:
    // const queryKey = `products:${category}-${page}`;

    // ... logic kiểm tra cache và fetch từ DB tương tự ví dụ 1 ...
    ```
*   **Lưu ý quan trọng:** Cần chuẩn hóa thứ tự và định dạng của các tham số truy vấn trước khi tạo key để tránh việc cùng một truy vấn nhưng tạo ra nhiều key cache khác nhau (ví dụ: `?a=1&b=2` và `?b=2&a=1` có thể tạo ra 2 key khác nhau nếu không chuẩn hóa).

### 4. Sử dụng TTL (Time-To-Live) và Cơ chế Hủy Cache (Cache Invalidation) hợp lý

*   **TTL (`EX` trong Redis `SET`):** Đặt thời gian sống cho dữ liệu cache. Redis sẽ tự động xóa key sau khoảng thời gian này.
    *   **Mục đích:** Đảm bảo dữ liệu cache không bao giờ trở nên quá cũ hoặc chiếm dụng bộ nhớ vô hạn.
    *   **Ví dụ:**
        ```javascript
        // Cache user:123 trong 5 phút (300 giây)
        await redis.set('user:123', JSON.stringify(user), 'EX', 300);
        // Hoặc dùng lệnh SETEX: await redis.setex('user:123', 300, JSON.parse(user)); // Lưu ý: SETEX không nhận object trực tiếp
        ```
*   **Cache Invalidation (`DEL`):** Xóa key cache một cách chủ động ngay khi dữ liệu gốc (trong DB) bị thay đổi (cập nhật, xóa).
    *   **Mục đích:** Đảm bảo tính nhất quán giữa dữ liệu trong DB và dữ liệu trong cache.
    *   **Ví dụ:**
        ```javascript
        // Giả sử user có ID 123 vừa cập nhật thông tin trong DB
        await db.updateUser(123, updatedData);

        // Ngay lập tức xóa cache của user này để lần sau sẽ fetch dữ liệu mới từ DB
        await redis.del('user:123');
        ```
*   **Cập nhật thời gian sống (`EXPIRE`):** Có thể cập nhật lại thời gian sống cho một key đã tồn tại mà không làm thay đổi giá trị của nó.

### 5. Sử dụng Pipeline hoặc `MGET` để Giảm Số Lần Roundtrip

*   **Vấn đề:** Mỗi lần gửi lệnh đến Redis (`.get()`, `.set()`, `.del()`) là một lần "roundtrip" mạng. Nếu cần thực hiện nhiều thao tác liên tiếp, tổng thời gian chờ roundtrip có thể đáng kể, đặc biệt khi Redis server không ở cùng máy chủ với ứng dụng.
*   **Giải pháp:**
    *   **`MGET`:** Lấy nhiều giá trị cùng lúc chỉ với một lệnh roundtrip.
        ```javascript
        // Lấy dữ liệu của 3 user cùng lúc
        const userIds = ['user:1', 'user:2', 'user:3'];
        const values = await redis.mget(userIds);
        // values sẽ là một mảng [json_user1, json_user2, json_user3] hoặc null cho key không tồn tại
        console.log(values.map(v => v ? JSON.parse(v) : null)); // Parse kết quả nếu có
        ```
    *   **Pipeline:** Nhóm nhiều lệnh ghi/đọc/xóa lại thành một lô (batch) và gửi đi cùng lúc. Redis xử lý các lệnh này tuần tự nhưng phản hồi lại một lần duy nhất sau khi tất cả hoàn thành.
        ```javascript
        const items = [ /* mảng các item cần cache */ ];
        const pipeline = redis.pipeline(); // Tạo một pipeline

        // Thêm các lệnh SET vào pipeline
        items.forEach(item => {
          pipeline.set(`item:${item.id}`, JSON.stringify(item), 'EX', 600);
        });

        // Thực thi tất cả các lệnh trong pipeline
        const results = await pipeline.exec();
        // results là một mảng chứa kết quả của từng lệnh trong pipeline
        console.log(results); // Mỗi phần tử có dạng [error, result]
        ```
*   **Mục đích:** Giảm đáng kể độ trễ (latency) do roundtrip mạng, đặc biệt hiệu quả khi thao tác với số lượng lớn key cùng lúc.

### 6. Cache Tầng 2 (Two-Level Cache)

*   **Ý tưởng:** Kết hợp Redis cache (tầng 1) với cache trong bộ nhớ (in-memory cache - tầng 2) ngay trong process của ứng dụng Node.js.
*   **Cơ chế:**
    1.  Request đến, kiểm tra cache Tầng 2 (in-memory) trước. Siêu nhanh vì không có roundtrip mạng.
    2.  Nếu không có trong Tầng 2, kiểm tra Tầng 1 (Redis). Nhanh hơn DB.
    3.  Nếu không có trong cả 2, fetch từ DB, sau đó lưu vào cả Tầng 1 và Tầng 2 cho các request sau.
*   **Mục đích:** Tăng tốc độ phản hồi cực đại cho các dữ liệu được truy cập lặp lại *liên tục trong cùng một instance*, đồng thời giảm tải cho cả Redis. Giúp bảo vệ Redis khỏi bị overload.
*   **Ví dụ (sử dụng thư viện `node-cache` đơn giản):**
    ```javascript
    const NodeCache = require("node-cache");
    const localCache = new NodeCache({ stdTTL: 30, checkperiod: 5 }); // Cache 30s, check hết hạn mỗi 5s

    async function getUserWithTwoLevelCache(userId) {
      const localKey = `user:${userId}`;
      const redisKey = `user:${userId}`;

      // Bước 1: Kiểm tra Local Cache (Tầng 2)
      let user = localCache.get(localKey);
      if (user) {
        console.log('Local cache hit!');
        return user; // Found in local cache, return immediately
      }

      // Bước 2: Kiểm tra Redis Cache (Tầng 1)
      console.log('Local cache miss, checking Redis...');
      const redisData = await redis.get(redisKey); // 'redis' là client Redis đã kết nối

      if (redisData) {
        console.log('Redis cache hit!');
        user = JSON.parse(redisData);
        // Nếu tìm thấy trong Redis, lưu vào Local Cache cho lần sau
        localCache.set(localKey, user); // TTL của local cache được set khi khởi tạo NodeCache
        return user;
      }

      // Bước 3: Cache miss ở cả 2 tầng, fetch từ DB
      console.log('Redis cache miss, fetching from DB...');
      user = await db.getUserById(userId); // 'db' là client DB đã kết nối

      if (user) {
        // Lưu vào cả Redis Cache (Tầng 1)
        await redis.set(redisKey, JSON.stringify(user), 'EX', 600); // Cache trên Redis 10 phút
        // Và lưu vào Local Cache (Tầng 2)
        localCache.set(localKey, user); // TTL của local cache được set khi khởi tạo NodeCache
      }

      return user;
    }

    // Cách sử dụng trong route handler:
    // app.get('/users/:id', async (req, res) => {
    //   const user = await getUserWithTwoLevelCache(req.params.id);
    //   if (user) {
    //     res.json(user);
    //   } else {
    //     res.status(404).send('User not found');
    //   }
    // });
    ```

### 7. Cache Dạng Danh Sách và Tập Hợp (Lists, Sets, Sorted Sets)

*   **Ý tưởng:** Redis không chỉ giới hạn ở việc cache các chuỗi (string). Bạn có thể sử dụng các cấu trúc dữ liệu nâng cao của Redis như Lists, Sets, và Sorted Sets để cache các danh sách, bảng xếp hạng, hoặc hàng đợi.
*   **Lợi ích:**
    *   Thao tác với danh sách/tập hợp ngay trên Redis siêu nhanh, không cần fetch toàn bộ data về ứng dụng rồi mới xử lý (ví dụ: lấy top N, lấy các phần tử trong khoảng index).
    *   Tiết kiệm bộ nhớ và băng thông.
*   **Ví dụ với Sorted Set (cho bảng xếp hạng/top list):**
    ```javascript
    // Thêm/cập nhật điểm cho sản phẩm bán chạy (score là số lượng bán được)
    // ZADD key score member [score member ...]
    await redis.zadd('top-products', 100, 'product:123');
    await redis.zadd('top-products', 85, 'product:456');
    await redis.zadd('top-products', 120, 'product:789');

    // Lấy top 10 sản phẩm bán chạy nhất (thứ tự giảm dần theo score)
    // ZREVRANGE key start stop [WITHSCORES]
    const topProducts = await redis.zrevrange('top-products', 0, 9, 'WITHSCORES');

    console.log('Top products (member, score):', topProducts);
    // Kết quả có thể dạng: ['product:789', '120', 'product:123', '100', ...]

    // --- Ví dụ với List (cho queue hoặc lịch sử gần đây) ---
    // Thêm item vào đầu danh sách (recent history)
    // LPUSH key element [element ...]
    await redis.lpush('recent-views:user:123', 'product:abc');
    await redis.lpush('recent-views:user:123', 'product:def');

    // Giới hạn kích thước danh sách (chỉ giữ 10 item gần nhất)
    // LTRIM key start stop
    await redis.ltrim('recent-views:user:123', 0, 9);

    // Lấy 5 item đầu tiên trong danh sách (5 sản phẩm xem gần đây nhất)
    // LRANGE key start stop
    const recentViews = await redis.lrange('recent-views:user:123', 0, 4);
    console.log('Recent views:', recentViews); // Kết quả (theo thứ tự LPUSH): ['product:def', 'product:abc']
    ```

## Kết Luận

Redis không chỉ đơn thuần là một "bộ nhớ tạm" (cache) cơ bản. Khi được sử dụng một cách thông minh và chiến lược, Redis có thể trở thành một công cụ mạnh mẽ để:

*   **Tăng tốc API:** Cải thiện đáng kể thời gian phản hồi của các API, có thể lên tới 10 lần hoặc hơn.
*   **Giảm chi phí hạ tầng:** Giảm tải cho database chính, cho phép xử lý lượng traffic lớn hơn với cùng một cấu hình DB hoặc giảm nhu cầu mở rộng DB.
*   **Giữ hệ thống ổn định:** Giúp hệ thống duy trì hiệu năng và ổn định ngay cả dưới mức tải cao.

**Tóm lại, để sử dụng Redis cache hiệu quả trong Node.js:**

*   **Cache đúng:** Xác định rõ loại dữ liệu cần cache, sử dụng key động (theo ID, query) và chỉ cache những dữ liệu phù hợp.
*   **Cache đủ:** Đặt thời gian sống (TTL) hợp lý và triển khai cơ chế hủy cache khi dữ liệu gốc thay đổi để đảm bảo tính tươi mới của dữ liệu.
*   **Cache hiệu quả:** Tận dụng các lệnh batch (`MGET`, Pipeline), cân nhắc sử dụng cache tầng 2 (in-memory) cho các trường hợp truy cập rất frequent trong cùng một instance, và sử dụng đúng cấu trúc dữ liệu của Redis (List, Sorted Set) cho từng nhu cầu cụ thể.

Áp dụng những kỹ thuật này sẽ giúp bạn tận dụng tối đa sức mạnh của Redis trong các ứng dụng Node.js của mình, xây dựng hệ thống nhanh hơn, hiệu quả hơn và có khả năng mở rộng tốt hơn.

Tuyệt Vời, đây là bản dịch và điều chỉnh nội dung bài viết thành định dạng README file bằng tiếng Việt:

---

# 2 Tất Cả Những Gì Bạn Cần Là Express và JSX
Sponsor by https://leapcell.io/blog/all-you-need-is-express-and-jsx?ref=dailydev

**Server-Side Rendering trong Express.js: So Sánh Chuyên Sâu EJS và JSX (Thực hành với TypeScript)**

*Bài viết gốc bởi:* Daniel Hayes
*Ngày đăng gốc:* 17 tháng 5, 2025

Node.js kết hợp với Express.js vẫn là sự kết hợp "vàng" để xây dựng các ứng dụng web hiệu quả. Khi cần cung cấp nội dung HTML động cho client, Express giới thiệu khái niệm "view engine". Trong nhiều năm, EJS (Embedded JavaScript) đã trở thành lựa chọn phổ biến nhờ sự đơn giản của nó. Tuy nhiên, kể từ khi React ra đời, JSX (JavaScript XML), với cách tiếp cận xây dựng giao diện người dùng (UI) dựa trên component, đã được các nhà phát triển ưa chuộng rộng rãi, và triết lý của nó cũng hoàn toàn áp dụng được cho server-side rendering.

Bài viết này đi sâu vào cách sử dụng EJS truyền thống và JSX hiện đại để triển khai server-side rendering (SSR) trong một ứng dụng Express.js được phát triển bằng TypeScript. Chúng ta sẽ so sánh ưu nhược điểm, phương pháp triển khai cụ thể của chúng, và thảo luận về cách triển khai ứng dụng đã xây dựng lên nền tảng đám mây một cách thuận tiện.

## 1. Giới thiệu: Server-Side Rendering và View Engine

Server-Side Rendering (SSR) là công nghệ mà toàn bộ trang HTML được tạo ra ở phía máy chủ và sau đó gửi đến client. Phương pháp này giúp cải thiện đáng kể tốc độ tải trang ban đầu và thân thiện với tối ưu hóa công cụ tìm kiếm (SEO). Express.js đơn giản hóa quá trình tạo HTML động thông qua cơ chế view engine của nó.

Trách nhiệm chính của view engine là kết hợp các file template với dữ liệu động và biên dịch chúng thành chuỗi HTML cuối cùng. Bản thân Express không đi kèm với một view engine cụ thể nào; các nhà phát triển có thể tự do chọn và cấu hình nó thông qua `app.set('view engine', 'engine_name')`.

## 2. EJS: Một Template Engine Kinh điển

### 2.1 Tổng quan và Các Tính năng Cốt lõi của EJS

Đúng như tên gọi, EJS (Embedded JavaScript) cho phép nhúng mã JavaScript vào các template HTML. Đối với các nhà phát triển quen thuộc với các ngôn ngữ script server-side truyền thống như PHP và ASP, cú pháp EJS rất trực quan và dễ hiểu.

Các thẻ EJS chính:

*   `<%= ... %>`: Hiển thị kết quả của một biểu thức JavaScript vào HTML (có thoát ký tự đặc biệt để tránh XSS).
*   `<%- ... %>`: Hiển thị kết quả của một biểu thức JavaScript vào HTML mà không thoát ký tự đặc biệt (dùng khi bạn *muốn* nhúng nội dung HTML).
*   `<% ... %>`: Dùng để thực thi các câu lệnh điều khiển JavaScript (như `if`, `for`,...).
*   `<%# ... %>`: Thẻ comment, nội dung không được thực thi hay hiển thị.
*   `<%- include('path/to/template') %>`: Nhúng và render một file EJS khác.

### 2.2 Sử dụng EJS trong Express (TypeScript)

**a. Cài đặt Dependencies:**

```bash
npm install express ejs
npm install --save-dev @types/express @types/ejs typescript nodemon ts-node
```

**b. Cấu hình `tsconfig.json` cơ bản:**

```json
{
  "compilerOptions": {
    "target": "ES2022", // Phiên bản JavaScript đích
    "module": "commonjs", // Hệ thống module phổ biến cho môi trường Node.js
    "rootDir": "./src",   // Thư mục chứa file TypeScript nguồn
    "outDir": "./dist",   // Thư mục xuất file JavaScript đã biên dịch
    "esModuleInterop": true, // Cho phép tương tác giữa CommonJS và ES Modules
    "strict": true,        // Bật tất cả các tùy chọn kiểm tra kiểu nghiêm ngặt
    "skipLibCheck": true   // Bỏ qua kiểm tra kiểu các file khai báo (.d.ts)
  },
  "include": ["src/**/*"], // Chỉ định các file cần biên dịch
  "exclude": ["node_modules"] // Chỉ định các file loại trừ khỏi biên dịch
}
```

**c. Ví dụ mã nguồn (`src/server.ts`):**

```typescript
import express, { Request, Response } from 'express';
import path from 'path';

const app = express();
const port = process.env.PORT || 3001; // Số cổng có thể tùy chỉnh

// Cấu hình EJS làm view engine
app.set('view engine', 'ejs');
// Cấu hình thư mục chứa file template, ví dụ: 'src/views'
app.set('views', path.join(__dirname, 'views'));

app.get('/', (req: Request, res: Response) => {
  res.render('index', { // Render views/index.ejs
    title: 'Trang Demo EJS',
    message: 'Chào mừng đến với template EJS được điều khiển bởi Express và TypeScript!',
    user: { name: 'Khách', isAdmin: false },
    items: ['Táo', 'Chuối', 'Cherry']
  });
});

app.listen(port, () => {
  console.log(`Máy chủ ví dụ EJS đang chạy tại http://localhost:${port}`);
});
```

**d. Ví dụ Template (`src/views/index.ejs`):**

```ejs
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= title %></title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    .user-greeting { color: blue; }
    .admin-panel { border: 1px solid red; padding: 10px; margin-top: 10px; }
  </style>
</head>
<body>
  <h1><%= message %></h1>

  <% if (user && user.name) { %>
    <p class="user-greeting">Xin chào, <%= user.name %>!</p>
    <% if (user.isAdmin) { %>
      <div class="admin-panel">Chào mừng, quản trị viên! Đây là bảng điều khiển admin.</div>
    <% } else { %>
      <p>Bạn hiện có quyền người dùng thông thường.</p>
    <% } %>
  <% } %>

  <h2>Danh sách Sản phẩm:</h2>
  <% if (items && items.length > 0) { %>
    <ul>
      <% items.forEach(function(item) { %>
        <li><%= item %></li>
      <% }); %>
    </ul>
  <% } else { %>
    <p>Không có sản phẩm nào.</p>
  <% } %>

  <%- include('partials/footer', { year: new Date().getFullYear() }) %>
</body>
</html>
```

**e. Ví dụ Partial Template (`src/views/partials/footer.ejs`):**

```ejs
<hr>
<footer>
  <p>&copy; <%= year %> Trang web của tôi. Bảo lưu mọi quyền.</p>
</footer>
```

### 2.3 Ưu điểm của EJS

*   **Đơn giản và Trực quan:** Đường cong học tập nhẹ nhàng, rất thân thiện với các nhà phát triển có kiến thức về HTML và JavaScript cơ bản.
*   **Linh hoạt cao:** Cho phép nhúng và thực thi trực tiếp mã JavaScript tùy ý trong template, mang lại sự tự do tương đối trong xử lý logic phức tạp.
*   **Sử dụng rộng rãi và Hệ sinh thái trưởng thành:** Là một template engine lâu đời, nó có số lượng lớn các dự án hiện có và sự hỗ trợ từ cộng đồng.

### 2.4 Nhược điểm của EJS

*   **Thiếu An toàn Kiểu (Type Safety):** Ngay cả khi dự án chính sử dụng TypeScript, dữ liệu truyền vào template EJS hầu như có kiểu `any` bên trong template. Điều này gây khó khăn trong việc phát hiện các vấn đề như lỗi chính tả tên thuộc tính hoặc sai lệch cấu trúc dữ liệu trong quá trình biên dịch, dễ xảy ra lỗi runtime.
*   **Thách thức về Khả năng đọc và Bảo trì:** Khi template nhúng quá nhiều hoặc logic JavaScript phức tạp, cấu trúc HTML và logic nghiệp vụ trở nên gắn kết chặt chẽ, khiến mã khó đọc và bảo trì.
*   **Khả năng Componentization yếu:** Mặc dù chỉ thị `include` có thể đạt được việc tái sử dụng các đoạn template, nhưng so với mô hình component dựa trên khai báo và khả năng kết hợp mà JSX cung cấp, EJS gặp khó khăn trong việc xây dựng các giao diện người dùng lớn và phức tạp.
*   **Hỗ trợ IDE hạn chế:** Trong các file `.ejs`, các tính năng mạnh mẽ của TypeScript như kiểm tra kiểu, gợi ý thông minh và refactoring không thể được tận dụng đầy đủ.

## 3. JSX: Một Mở rộng Cú pháp để Xây dựng UI

### 3.1 Tổng quan và Các Tính năng Cốt lõi của JSX (Không chỉ dành cho React)

JSX (JavaScript XML) là một mở rộng cú pháp cho JavaScript cho phép các nhà phát triển viết cấu trúc giống HTML (hoặc XML) trong mã JavaScript. Mặc dù ban đầu được thiết kế cho React, bản thân JSX là một đặc tả độc lập có thể được biên dịch thành bất kỳ mã đích nào, không chỉ dành riêng cho React. Ở phía máy chủ, chúng ta có thể tận dụng các tính năng khai báo của JSX để mô tả cấu trúc UI và sau đó chuyển đổi chúng thành chuỗi HTML.

Mã JSX không thể được thực thi trực tiếp bởi trình duyệt hoặc môi trường Node.js; nó cần được transpiled thành các lời gọi hàm JavaScript chuẩn (như `React.createElement()` hoặc các hàm tương đương tùy chỉnh) thông qua các công cụ như Babel, TypeScript compiler (tsc), hoặc esbuild.

### 3.2 Tại sao chọn JSX cho Server-Side Rendering?

Đối với các nhà phát triển đã quen thuộc với các framework frontend hiện đại như React và Vue, JSX (hoặc cú pháp template tương tự) là lựa chọn tự nhiên để xây dựng UI theo hướng component hóa. Việc giới thiệu nó vào server-side rendering mang lại nhiều lợi ích:

*   **Tính nhất quán giữa Frontend và Backend:** Cho phép tư duy thiết kế và mô hình phát triển component hóa tương tự trên cả phía máy chủ và client.
*   **An toàn Kiểu (Type Safety):** Kết hợp với TypeScript, nó cho phép định nghĩa kiểu rõ ràng cho Props (thuộc tính) của component, tận hưởng sự mạnh mẽ từ việc kiểm tra kiểu tại thời điểm biên dịch.
*   **Khai báo (Declarative) và Có cấu trúc:** Mã UI mang tính khai báo hơn, với cấu trúc rõ ràng hơn, giúp dễ hiểu và bảo trì hơn.
*   **Tái sử dụng Component:** Tạo và tái sử dụng các UI component dễ dàng, cải thiện hiệu quả phát triển.

### 3.3 Cấu hình Môi trường JSX trong Express (TypeScript)

Để sử dụng JSX cho server-side rendering trong Express, cần một số cấu hình. Chúng ta sẽ sử dụng `react` và `react-dom/server` để chuyển đổi các component JSX thành chuỗi HTML. Lưu ý rằng điều này khác với React phía client; ở đây, chúng ta chỉ tận dụng khả năng phân tích cú pháp JSX và tạo chuỗi của nó, không liên quan đến các hoạt động của Virtual DOM hay lifecycle phía client.

**a. Cài đặt Dependencies cần thiết:**

```bash
npm install express react react-dom
npm install --save-dev @types/express @types/react @types/react-dom typescript nodemon ts-node esbuild esbuild-register
```

`esbuild` là một công cụ bundling và transpiling JavaScript/TypeScript cực nhanh. `esbuild-register` có thể ngay lập tức transpile các file `.ts` và `.tsx` trong quá trình phát triển, rất tiện lợi. Trong môi trường production, thường khuyến nghị biên dịch trước.

**b. Cấu hình `tsconfig.json`:**

Để đảm bảo trình biên dịch TypeScript xử lý đúng cú pháp JSX, cần cấu hình sau trong `tsconfig.json`:

```json
{
  "compilerOptions": {
    // ... Các cấu hình khác giữ nguyên ...
    "jsx": "react-jsx", // Khuyến nghị cho chuyển đổi JSX mới, không cần import React thủ công
    // "jsx": "react", // Chuyển đổi JSX cũ, yêu cầu import React trong mỗi file .tsx: import React from 'react';
    "esModuleInterop": true,
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

Tùy chọn `"jsx": "react-jsx"` bật chuyển đổi JSX mới, tự động import các hàm trợ giúp cần thiết. Thông thường, không cần viết `import React from 'react';` ở đầu mỗi file JSX (trừ khi bạn sử dụng rõ ràng các API React khác như Hooks, thường không cần thiết trong các component SSR thuần túy).

**c. Triển khai Custom JSX View Engine:**

Express yêu cầu một view engine tùy chỉnh để xử lý các file `.tsx`. Engine này chịu trách nhiệm `require` (hoặc import động) các file `.tsx` đã được biên dịch (tức là file `.js`), lấy các component được export và sử dụng `ReactDOMServer.renderToString()` để chuyển đổi chúng cùng với `props` thành chuỗi HTML.

Cấu hình trong `src/server.tsx` (hoặc `src/app.ts`):

```typescript
// src/server.tsx or src/app.ts
import express, { Request, Response, NextFunction } from 'express';
import path from 'path';
import ReactDOMServer from 'react-dom/server'; // Dùng cho server-side rendering
import React from 'react'; // Vẫn cần import React cho các type và React.createElement
import fs from 'fs'; // Module hệ thống file của Node.js

// Ở chế độ phát triển, sử dụng esbuild-register để biên dịch file .ts/.tsx tức thời
// Ở chế độ production, biên dịch trước src sang thư mục dist và chạy các file .js từ dist
if (process.env.NODE_ENV !== 'production') {
  require('esbuild-register')({
    extensions: ['.ts', '.tsx'], // Đảm bảo xử lý file .tsx
    // target: 'node16' // Đặt theo phiên bản Node.js của bạn
  });
}

const app = express();
const port = process.env.PORT || 3002;

// Custom JSX (.tsx) view engine
app.engine('tsx', async (filePath: string, options: object, callback: (e: any, rendered?: string) => void) => {
  try {
    // Import module đã biên dịch động (được xử lý bởi esbuild-register hoặc tsc)
    // Lưu ý: Cơ chế cache của require có thể ảnh hưởng đến cập nhật nóng; không có vấn đề này trong production builds
    // Để cập nhật nóng tin cậy hơn trong phát triển, có thể cần thiết lập phức tạp hơn như xóa require.cache
    // delete require.cache[require.resolve(filePath)]; // Ví dụ xóa cache đơn giản, có thể có tác dụng phụ
    const { default: Component } = await import(filePath); // Giả định component là default export

    if (!Component) {
      return callback(new Error(`Component không tìm thấy hoặc không default-exported từ ${filePath}`));
    }

    // Sử dụng API của React để render component thành chuỗi HTML
    // Đối tượng options được truyền vào component dưới dạng props
    const html = ReactDOMServer.renderToString(React.createElement(Component, options));

    // Thường cần bọc thêm <!DOCTYPE html>
    callback(null, `<!DOCTYPE html>${html}`);
  } catch (e) {
    callback(e);
  }
});

app.set('views', path.join(__dirname, 'views')); // Cấu hình thư mục file view, ví dụ: 'src/views'
app.set('view engine', 'tsx'); // Cấu hình .tsx làm view engine mặc định

// ... Routes sẽ được định nghĩa sau ...
```

**Lưu ý:**

*   `await import(filePath)` được sử dụng cho import động. Trong hệ thống module `commonjs`, điều này thường hoạt động bình thường sau khi xuất sang thư mục `dist`. `esbuild-register` cũng xử lý tốt điều này.
*   **Hot Reloading trong Phát triển:** `require` hoặc `import` đơn giản sẽ bị Node.js cache lại. Để thay đổi component có hiệu lực ngay lập tức trong quá trình phát triển, có thể cần thêm cơ chế hot module replacement (HMR) hoặc xóa cache thủ công (`require.cache`) (như trong các comment ví dụ, nhưng không khuyến nghị cho các tình huống phức tạp). Production builds không gặp vấn đề này.
*   `React.createElement(Component, options)` là cách sử dụng chuẩn hơn so với `Component(options)`, mặc dù cách sau có thể hoạt động trong các kịch bản đơn giản.

### 3.4 Tạo JSX Components

Tạo các component JSX (các file `.tsx`) trong thư mục `src/views`.

**a. Layout Component:**

Tạo một component layout trang chung để bọc toàn bộ nội dung trang và cung cấp một bộ khung HTML nhất quán.

`src/views/layouts/MainLayout.tsx`:

```typescript
// Với "jsx": "react-jsx" được đặt trong tsconfig.json, việc import React từ 'react' thường không cần thiết
// Nhưng nếu sử dụng các API React cụ thể (như createContext, useState, ...), vẫn cần import
// import React from 'react';

interface MainLayoutProps {
  title: string;
  children: React.ReactNode; // Để nhận các component con
  lang?: string;
}

// React.FC (Functional Component) là một kiểu tùy chọn cung cấp một số tiện ích
const MainLayout: React.FC<MainLayoutProps> = ({ title, children, lang = "vi" }) => {
  return (
    <html lang={lang}>
      <head>
        <meta charSet="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>{title}</title>
        {/* Các link CSS global, meta tags, ... có thể thêm ở đây */}
        <link rel="stylesheet" href="/styles/global.css" /> {/* Giả định có stylesheet global */}
        <style dangerouslySetInnerHTML={{ __html: `
          body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 0; padding: 20px; background-color: #f4f4f4; color: #333; }
          header { background-color: #333; color: white; padding: 1rem; text-align: center; }
          main { background-color: white; padding: 1rem; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
          footer { text-align: center; margin-top: 2rem; color: #666; }
        `}} />
      </head>
      <body>
        <header>
          <h1>{title}</h1>
        </header>
        <main>
          {children} {/* Các component con sẽ được render ở đây */}
        </main>
        <footer>
          <p>&copy; {new Date().getFullYear()} Leapcell Technical Demo</p>
        </footer>
      </body>
    </html>
  );
};

export default MainLayout;
```

**Quan trọng:** Khi sử dụng `React.FC`, bắt đầu từ `@types/react` v18, `children` không còn là thuộc tính ngầm định và phải được khai báo rõ ràng trong kiểu Props dưới dạng `children: React.ReactNode;`.

**b. Page Component:**

Tạo các component cho các trang cụ thể.

`src/views/pages/HomePage.tsx`:

```typescript
import React from 'react'; // Vẫn cần import React cho các type
import MainLayout from '../layouts/MainLayout'; // Import component layout

interface HomePageProps {
  pageTitle: string;
  welcomeMessage: string;
  features: Array<{ id: number; name: string; description: string }>;
  currentUser?: string; // Thuộc tính tùy chọn
}

const HomePage: React.FC<HomePageProps> = (props) => {
  const { pageTitle, welcomeMessage, features, currentUser } = props;

  return (
    <MainLayout title={pageTitle}>
      <h2>{welcomeMessage}</h2>
      {currentUser && <p>Người dùng hiện tại: <strong>{currentUser}</strong></p>}

      <h3>Các tính năng cốt lõi:</h3>
      {features.length > 0 ? (
        <ul>
          {features.map(feature => (
            <li key={feature.id}>
              <strong>{feature.name}:</strong> {feature.description}
            </li>
          ))}
        </ul>
      ) : (
        <p>Không có giới thiệu tính năng nào.</p>
      )}

      <div style={{ marginTop: '20px', padding: '10px', border: '1px dashed #ccc' }}>
        <p>Đây là khu vực ví dụ với style inline.</p>
      </div>
    </MainLayout>
  );
};
export default HomePage; // Đảm bảo export default
```

### 3.5 Rendering JSX trong Express Routes

Quay lại `src/server.tsx` (hoặc `src/app.ts`) và thêm các route để render các component trang JSX đã tạo.

```typescript
// src/server.tsx (hoặc src/app.ts) (tiếp theo từ trên)

// Ví dụ: Middleware phục vụ tài nguyên tĩnh cho CSS, JS, hình ảnh, v.v.
// Giả định bạn đã tạo một stylesheet tại src/public/styles/global.css
app.use(express.static(path.join(__dirname, 'public')));
// Lưu ý: Nếu __dirname trỏ đến dist/src, thư mục public cần được điều chỉnh cho phù hợp
// Thông thường, thư mục public được đặt ở thư mục gốc dự án hoặc ngang hàng với src, và được sao chép sang dist trong quá trình build

app.get('/', (req: Request, res: Response) => {
  const homePageData = {
    pageTitle: 'Trang chủ SSR với JSX',
    welcomeMessage: 'Chào mừng đến với trải nghiệm Express + TypeScript + JSX server-side rendering!',
    features: [
      { id: 1, name: 'An toàn kiểu', description: 'Hỗ trợ kiểu mạnh mẽ cho Props thông qua TypeScript.' },
      { id: 2, name: 'Componentization', description: 'Chia UI thành các component có thể tái sử dụng.' },
      { id: 3, name: 'Trải nghiệm phát triển hiện đại', description: 'Tận hưởng mô hình phát triển nhất quán với các framework frontend hiện đại.' }
    ],
    currentUser: 'Khám phá Leap'
  };
  res.render('pages/HomePage', homePageData); // Render views/pages/HomePage.tsx
});

app.get('/info', (req: Request, res: Response) => {
  // Giả định bạn đã tạo src/views/pages/InfoPage.tsx
  // res.render('pages/InfoPage', { /* ... props ... */ });

  // Ví dụ đơn giản: Trả về trực tiếp chuỗi HTML (không khuyến nghị, chỉ để minh họa)
  // Cách tốt hơn: Tạo một component .tsx cho InfoPage
  const InfoComponent = () => (
    <MainLayout title="Trang thông tin">
      <p>Đây là một trang thông tin đơn giản cũng được render bởi JSX ở phía máy chủ.</p>
      <p>Thời gian hiện tại: {new Date().toLocaleTimeString()}</p>
    </MainLayout>
  );
  const html = ReactDOMServer.renderToString(<InfoComponent />);
  res.send(`<!DOCTYPE html>${html}`);

});

app.listen(port, () => {
  console.log(`Máy chủ ví dụ JSX đang chạy tại http://localhost:${port}`);
  if (process.env.NODE_ENV !== 'production') {
    console.log('Chế độ phát triển: Sử dụng esbuild-register để biên dịch TSX tức thời.');
  } else {
    console.log('Chế độ production: Đảm bảo bạn đang chạy các file JS đã biên dịch trước từ thư mục dist.');
  }
});
```

Bây giờ, khi truy cập route `/`, Express sẽ sử dụng engine `tsx` mà chúng ta đã định nghĩa để tải component `HomePage.tsx`, truyền `homePageData` làm props, và render nó thành chuỗi HTML để trả về trình duyệt.

### 3.6 Ưu điểm của JSX trong SSR

*   **An toàn Kiểu mạnh mẽ:** Sự kết hợp giữa TypeScript và JSX là hoàn hảo. Bạn có thể định nghĩa các giao diện (`interface`) hoặc kiểu (`type`) chính xác cho Props của component, và trình biên dịch sẽ bắt lỗi kiểu không khớp, thiếu thuộc tính hoặc lỗi chính tả trong quá trình phát triển, giúp tăng cường đáng kể độ mạnh mẽ và khả năng bảo trì mã.
*   **Khả năng Componentization xuất sắc:** UI có thể được chia rõ ràng thành các component có độ kết dính cao và khớp nối lỏng. Các component này không chỉ dễ hiểu và kiểm thử mà còn có thể được tái sử dụng trên các trang khác nhau hoặc thậm chí giữa các dự án. Các khái niệm như layout component và atomic component có thể dễ dàng được triển khai.
*   **Cải thiện Trải nghiệm Nhà phát triển (DX):**
    *   **Hỗ trợ IDE:** Các IDE phổ biến (ví dụ: VS Code) cung cấp hỗ trợ tuyệt vời cho TSX, bao gồm tự động hoàn thành mã thông minh, gợi ý kiểu, refactoring, đánh dấu lỗi, v.v.
    *   **Lập trình Khai báo:** Cú pháp khai báo của JSX làm cho cấu trúc UI trực quan hơn, mã gần gũi hơn với giao diện hiển thị cuối cùng.
    *   **Tích hợp Hệ sinh thái:** Có thể tận dụng một số thư viện hoặc design pattern độc lập với rendering trong hệ sinh thái React.
    *   **Nhất quán mã:** Nếu frontend cũng sử dụng React hoặc các framework dựa trên JSX tương tự, server và client có thể sử dụng phong cách viết component và logic tương tự, giảm chi phí học tập và chuyển đổi ngữ cảnh cho các thành viên trong nhóm.

### 3.7 Các Thách thức tiềm ẩn của JSX trong SSR

*   **Cấu hình Build:** JSX yêu cầu bước biên dịch (TypeScript compiler, Babel, hoặc esbuild) để chuyển đổi thành JavaScript có thể thực thi cho trình duyệt hoặc Node.js. Mặc dù `esbuild-register` đơn giản hóa quá trình phát triển, việc triển khai production vẫn cần một chiến lược build hợp lý.
*   **Chi phí hiệu năng nhỏ:** So với nối chuỗi trực tiếp hoặc các template engine rất nhẹ, việc biên dịch JSX và các lời gọi `ReactDOMServer.renderToString()` giới thiệu một số chi phí hiệu năng. Tuy nhiên, trong hầu hết các kịch bản ứng dụng, chi phí này thường không đáng kể và có thể được tối ưu hóa thông qua các chiến lược caching.
*   **Mô hình tư duy:** Các nhà phát triển chưa quen với React hoặc JSX cần một thời gian học tập để làm quen với tư duy component hóa và mô hình lập trình hàm của nó.
*   **Giới hạn phía Server:** Trong các kịch bản SSR thuần túy, React Hooks (ví dụ: `useState`, `useEffect`) thường không có ý nghĩa vì server-side rendering là một quá trình một lần, không liên quan đến tương tác phía client hoặc cập nhật trạng thái. Các component chủ yếu nên dựa vào `props` để nhận dữ liệu và render.

## 4. EJS vs. JSX: So sánh Toàn diện

| Tính năng               | EJS (Embedded JavaScript)                          | JSX (với TypeScript)                                     |
| :---------------------- | :------------------------------------------------- | :------------------------------------------------------- |
| Cú pháp và Khả năng đọc | HTML nhúng JS (`<% ... %>`). Logic đơn giản rõ ràng; logic phức tạp trở nên lộn xộn. | JS nhúng cấu trúc giống HTML. Component hóa, cấu trúc UI phức tạp rõ ràng hơn. |
| An toàn kiểu (Type Safety) | Yếu. Dữ liệu truyền vào template gần như không có kiểm tra kiểu, dễ xảy ra lỗi runtime. | Mạnh mẽ. Props được định kiểu qua TypeScript, kiểm tra tại thời điểm biên dịch, rất mạnh mẽ. |
| Componentization và Tái sử dụng | Hạn chế (`include`). Khó đạt được componentization thực sự và cách ly trạng thái. | Tính năng cốt lõi. Hỗ trợ native cho các component có khả năng tái sử dụng và kết hợp cao. |
| Trải nghiệm nhà phát triển (DX) | Hỗ trợ IDE hạn chế, debug tương đối khó, refactoring bất tiện. | Hỗ trợ IDE mạnh mẽ (gợi ý thông minh, kiểm tra kiểu, refactoring), debug thân thiện. |
| Hiệu năng                | Thường rất nhẹ, phân tích nhanh.                     | Biên dịch và `renderToString` có một số chi phí, nhưng chấp nhận được trong hầu hết các kịch bản. |
| Hệ sinh thái và Công cụ | Đơn giản, ít dependency.                             | Dựa vào các thư viện liên quan đến React và công cụ biên dịch (tsc, esbuild, Babel). |
| Xử lý Logic             | Cho phép viết logic JS phức tạp trực tiếp trong template (không khuyến nghị). | Khuyến nghị đặt logic trong các phương thức/hooks của component (nếu áp dụng) hoặc props truyền vào. |

## 5. Khuyến nghị Lựa chọn Công nghệ: Khi nào chọn EJS hay JSX?

**Các kịch bản nên chọn EJS:**

*   **Các dự án rất đơn giản:** Ít trang, logic UI không phức tạp, mục tiêu là prototyping nhanh.
*   **Đội ngũ chưa quen với React/JSX:** Thiếu thời gian hoặc mong muốn học tập.
*   **Hạn chế nghiêm ngặt về các bước build:** Muốn giảm thiểu các giai đoạn biên dịch.
*   **Bảo trì dự án cũ:** Cơ sở template EJS lớn hiện có, chi phí chuyển đổi cao.

**Các kịch bản nên chọn JSX (với TypeScript):**

*   **Theo đuổi độ mạnh mẽ và khả năng bảo trì cao:** An toàn kiểu là yếu tố chính.
*   **Xây dựng UI phức tạp và có khả năng mở rộng:** Yêu cầu khả năng component hóa mạnh mẽ.
*   **Đội ngũ đã quen với React/JSX hoặc đang áp dụng các stack công nghệ frontend hiện đại:** Cải thiện hiệu quả phát triển và chất lượng mã.
*   **Nhất quán stack công nghệ Frontend-Backend:** Duy trì sự nhất quán kỹ thuật nếu frontend cũng sử dụng React.
*   **Các dự án vừa và lớn:** Lợi ích dài hạn từ componentization và an toàn kiểu vượt xa chi phí đầu tư ban đầu.

Tóm lại, đối với các dự án server-side rendering Node.js mới có một chút phức tạp, rất khuyến nghị sử dụng JSX với TypeScript. Những cải thiện về an toàn kiểu, khả năng componentization và trải nghiệm phát triển có thể nâng cao đáng kể chất lượng dự án và khả năng bảo trì dài hạn.

## 6. Triển khai Ứng dụng lên Nền tảng Đám mây (ví dụ: Leapcell)

Dù được xây dựng bằng EJS hay JSX hiện đại, việc triển khai một ứng dụng Express lên nền tảng đám mây khá đơn giản. Các nền tảng hosting đám mây hiện đại như Leapcell cung cấp trải nghiệm triển khai và quản lý thuận tiện cho các ứng dụng Node.js.

Quy trình triển khai điển hình bao gồm:

1.  **Chuẩn bị mã nguồn:** Đảm bảo file `package.json` định nghĩa một script `start`, ví dụ:
    ```json
    "scripts": {
      "start": "node dist/server.js", // Bắt đầu các file JS đã biên dịch trong production
      "build": "tsc" // Hoặc dùng esbuild: "esbuild src/server.tsx --bundle --outfile=dist/server.js --platform=node --format=cjs"
    }
    ```
2.  **Build:** Thực thi lệnh build (ví dụ: `npm run build`) để tạo thư mục `dist` trước khi triển khai.
3.  **Cấu hình nền tảng:** Trên các nền tảng như Leapcell:
    *   Kết nối với kho mã nguồn của bạn (ví dụ: GitHub).
    *   Cấu hình lệnh build (nếu nền tảng hỗ trợ auto-building, nó thường đọc script `build` từ `package.json`).
    *   Cấu hình lệnh start (ví dụ: `npm start` hoặc trực tiếp `node dist/server.js`).
    *   Cấu hình các biến môi trường (ví dụ: `PORT`, `NODE_ENV=production`, chuỗi kết nối database, v.v.).
4.  **Triển khai:** Nền tảng sẽ tự động kéo mã nguồn, thực thi build (nếu cấu hình), cài đặt dependencies và chạy ứng dụng của bạn theo lệnh start.

Các nền tảng như Leapcell thường cũng cung cấp các tính năng như xem log, tự động co giãn (auto-scaling), tên miền tùy chỉnh, HTTPS, v.v., cho phép nhà phát triển tập trung hơn vào triển khai logic nghiệp vụ thay vì các hoạt động máy chủ cơ bản. Quy trình này được chuẩn hóa cho các ứng dụng Express (dù sử dụng view engine EJS hay JSX).

## 7. Kết luận

Khi thực hiện server-side rendering trong Express.js, EJS và JSX đại diện cho hai mô hình phát triển khác nhau. EJS vẫn có vị trí của nó trong các dự án nhỏ nhờ sự đơn giản, nhưng JSX, với khả năng an toàn kiểu mạnh mẽ (khi kết hợp với TypeScript), khả năng componentization và trải nghiệm phát triển xuất sắc, chắc chắn là lựa chọn tốt hơn để xây dựng các ứng dụng web hiện đại, mạnh mẽ và dễ bảo trì.

Mặc dù việc giới thiệu JSX đòi hỏi một số cấu hình bổ sung và hiểu biết về hệ sinh thái React, lợi ích dài hạn của nó là đáng kể. Đối với các đội ngũ muốn cải thiện chất lượng mã và hiệu quả phát triển, việc áp dụng JSX (TSX) cho server-side rendering là một quyết định sáng suốt. Cuối cùng, bất kể công nghệ nào được chọn, các ứng dụng đều có thể dễ dàng được triển khai lên các nền tảng đám mây như Leapcell để tận hưởng các dịch vụ hosting ứng dụng hiện đại.

---