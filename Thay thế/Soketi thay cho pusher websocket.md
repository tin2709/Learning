Dựa trên nội dung bạn cung cấp, dưới đây là bản **README.md** chi tiết bằng tiếng Việt dành cho dự án **Soketi**. Tài liệu này được biên soạn lại để làm nổi bật các tính năng, hướng dẫn cài đặt và so sánh hiệu quả chi phí.

---

# Soketi - Máy chủ WebSockets Mã Nguồn Mở



<p align="center">
    <a href="https://github.com/soketi/soketi/actions"><img src="https://img.shields.io/github/workflow/status/soketi/soketi/CI" alt="Build Status"></a>
    <a href="https://www.npmjs.com/package/@soketi/soketi"><img src="https://img.shields.io/npm/v/@soketi/soketi" alt="NPM Version"></a>
    <a href="https://hub.docker.com/r/quay.io/soketi/soketi"><img src="https://img.shields.io/docker/pulls/quay.io/soketi/soketi" alt="Docker Pulls"></a>
    <a href="https://discord.gg/39dT3XN"><img src="https://img.shields.io/discord/706509636254826546?color=7289DA&label=discord" alt="Discord"></a>
    <a href="https://github.com/soketi/soketi/blob/master/LICENSE"><img src="https://img.shields.io/github/license/soketi/soketi" alt="License"></a>
</p>

**Soketi** là một máy chủ WebSockets mã nguồn mở, đơn giản, nhanh chóng và bền bỉ. Nó hoàn toàn tương thích với giao thức **Pusher v7**, cho phép bạn triển khai các ứng dụng thời gian thực (real-time) với chi phí thấp và hiệu năng cao.

---

## 🚀 Tại sao nên chọn Soketi?

### ⚡ Tốc độ siêu nhanh (Blazing Fast)
Soketi được xây dựng trên **uWebSockets.js** - một ứng dụng C được port sang Node.js. Nó được đảm bảo phân phối tin nhắn cốt lõi trong dưới **100ms**.
*   Hiệu suất gấp **8.5x** so với Fastify.
*   Hiệu suất gấp **10x** so với Socket.IO.
*   **6ms**: Thời gian trung bình để phân phối tin nhắn tới 1.000 người dùng chỉ với 200m CPU/200 MB RAM.

### 🤑 Tiết kiệm chi phí tối đa
Ngừng trả tiền cho các dịch vụ realtime đắt đỏ. Với Soketi, bạn chỉ phải trả tiền cho cơ sở hạ tầng của mình.
*   Chạy tốt trên các gói VPS $5/tháng (DigitalOcean, Vultr...).
*   Không giới hạn kết nối (so với giới hạn 500 kết nối của gói $49 Pusher).
*   Không giới hạn số lượng tin nhắn.

### 📡 Tương thích hoàn toàn với Pusher
Bạn không cần viết lại code. Soketi sử dụng **Pusher Protocol v7**.
*   Chỉ cần thay đổi thông tin kết nối (Host/Port) trong Client SDK.
*   Hỗ trợ Plug-and-play cho các dự án hiện có.

### 🤿 Sẵn sàng cho Production
*   **Mở rộng (Scaling):** Hỗ trợ mở rộng lên hàng trăm server sử dụng Redis hoặc NATS mà không làm giảm hiệu suất.
*   **Quản lý ứng dụng:** Tích hợp sẵn quản lý thông tin đăng nhập (App ID, Key, Secret) giống như Pusher. Hỗ trợ đọc từ Database (MySQL, DynamoDB, Postgres...).
*   **Webhooks:** Kích hoạt HTTP request khi kênh có hoạt động. Tương thích tuyệt vời với AWS Lambda hoặc Cloudflare Workers.

### ☁️ Soketi Serverless & Cloudflare
Soketi hiện đang trong giai đoạn Open Beta cho **Serverless**. Bạn có thể triển khai Soketi trực tiếp lên **Cloudflare Workers**. Mang WebSockets đến gần người dùng hơn với mạng lưới toàn cầu của Cloudflare.

---

## 📊 Bảng so sánh chi phí và tính năng

| Tính năng | Pusher | Ably | Soketi | Serverless (Cloudflare) |
| :--- | :--- | :--- | :--- | :--- |
| **Giá** | $49/tháng | $49/tháng | **~$5/tháng** (VPS) | ~$12/tháng* |
| **Kết nối tối đa** | 500 | 500 | **Không giới hạn** | **Không giới hạn** |
| **Tin nhắn/tháng** | 30 Triệu | 6 Triệu | **Không giới hạn** | **Không giới hạn** |
| **Dung lượng tin nhắn** | 10 KiB | 64 KiB | **Cấu hình tùy ý** | **Cấu hình tùy ý** |
| **Kênh tối đa** | Không giới hạn | Không giới hạn | **Không giới hạn** | **Không giới hạn** |
| **Mã nguồn mở** | ❌ | ❌ | ✅ **Có** | ❌ (Mã đóng, deploy qua Dashboard) |
| **On-premise (Tự host)**| ❌ | ❌ | ✅ **Có** | N/A |

---

## 🛠 Hướng dẫn cài đặt nhanh

Bạn có thể chạy Soketi ngay lập tức bằng Docker.

### Chạy Server
Sử dụng lệnh sau để khởi chạy server trên cổng `6001`:

```bash
docker run -p 6001:6001 quay.io/soketi/soketi:latest-16-alpine
```

Sau khi chạy, bạn sẽ thấy log:
```text
🕵️‍♂️ Initiating metrics endpoints...
🎉 Server is up and running!
📡 The Websockets server is available at 127.0.0.1:6001
🔗 The HTTP API server is available at http://127.0.0.1:6001
```

### Kết nối từ phía Client (Ví dụ Javascript)
Bạn sử dụng thư viện `pusher-js` như bình thường, chỉ cần trỏ host về server Soketi của bạn.

```javascript
import Pusher from 'pusher-js';

let client = new Pusher('app-key', {
  wsHost: '127.0.0.1',
  wsPort: 6001,
  forceTLS: false,
  encrypted: false, // Đặt là true nếu dùng SSL (wss)
  disableStats: true,
  enabledTransports: ['ws', 'wss'],
});

let channel = client.subscribe('chat-room');

channel.bind('message', (data) => {
  console.log('Nhận tin nhắn:', data);
});
```

---

## 🔗 Webhooks & Serverless
Soketi hỗ trợ Webhooks để kích hoạt các hàm xử lý (như AWS Lambda) khi có sự kiện xảy ra trên kênh.

Ví dụ xử lý webhook (Node.js):
```javascript
import { createHmac } from 'crypto';

exports.handler = async ({ payload, headers }) => {
  // Xác thực chữ ký để đảm bảo request đến từ Soketi
  let hmac = createHmac('sha256', process.env.SOKETI_SECRET)
    .update(JSON.stringify(payload))
    .digest('hex');

  let receivedSignature = headers['X-Pusher-Signature'] || null;

  if (receivedSignature !== hmac) {
    return; // Chữ ký không khớp
  }

  payload.events.forEach(({ name, channel }) => {
    if (name === 'channel_occupied') {
      console.log(`${channel} hiện đang có người tham gia.`);
    }
  });
};
```

---

## 💬 Mọi người nói gì về Soketi?

> "Soketi thật tuyệt vời! Tôi đã thay thế Pusher và chỉ mất 5 phút để triển khai. Đây là sự thay thế tuyệt vời cho các giải pháp đắt tiền."
> <br>— **Philo Hermans**, Founder / Unlock

> "Soketi giúp việc thiết lập một server Pusher tự host trở nên dễ dàng, bao gồm cả webhooks và hỗ trợ nhiều ứng dụng. Cực nhanh và dễ dàng mở rộng."
> <br>— **Alex Bouma**, Developer / @stayallive

> "Chúng tôi là fan lớn của Soketi. Hiện tại chúng tôi đang mở rộng lên đến 200k kết nối, thật xuất sắc."
> <br>— **Lawrence Dudley**, Director / Parallax

---

## 📚 Tài liệu & Liên kết hữu ích

*   **Trang chủ & Tài liệu:** [Documentation](https://docs.soketi.app/)
*   **Mã nguồn (GitHub):** [soketi/soketi](https://github.com/soketi/soketi)
*   **Docker Hub:** [quay.io/soketi/soketi](https://quay.io/repository/soketi/soketi)
*   **Cộng đồng Discord:** [Tham gia ngay](https://discord.gg/39dT3XN)
