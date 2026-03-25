Dưới đây là nội dung bài viết được chuyển đổi sang định dạng `README.md` chuyên nghiệp, tối ưu cho việc lưu trữ và chia sẻ kiến thức kỹ thuật.

---

# API Overclocking: Từ 200ms xuống 10ms với Kiến trúc Lai Bun & Node.js

![Performance](https://img.shields.io/badge/Performance-10ms-brightgreen)
![Runtime](https://img.shields.io/badge/Runtime-Bun%20%2B%20Node.js-blue)
![Architecture](https://img.shields.io/badge/Architecture-Hybrid--Worker-orange)

Tài liệu này hướng dẫn cách tối ưu hóa hiệu năng API cực hạn mà không cần viết lại toàn bộ logic nghiệp vụ cũ (legacy). Bằng cách kết hợp tốc độ I/O của Bun và tính ổn định của hệ sinh thái Node.js.

## 📖 Mục lục
- [1. Mô hình Tiền trạm (Bun) - Hậu phương (Node)](#1-mô-hình-tiền-trạm-bun---hậu-phương-node)
- [2. Tối ưu I/O: Zero-Copy với Bun](#2-tối-ưu-io-zero-copy-với-bun)
- [3. Micro-batching: Xếp hàng xử lý](#3-micro-batching-xếp-hàng-xử-lý)
- [4. Quản lý bộ nhớ & Garbage Collection](#4-quản-lý-bộ-nhớ--garbage-collection)
- [5. Chiến lược Cache 2 lớp](#5-chiến-lược-cache-2-lớp)
- [6. Native API vs NPM Bloat](#6-native-api-vs-npm-bloat)
- [7. Quản lý môi trường phát triển](#7-quản-lý-môi-trường-phát-triển)

---

## 1. Mô hình Tiền trạm (Bun) - Hậu phương (Node)

Sử dụng Bun làm lớp HTTP siêu mỏng (Gateway) để định tuyến và validate. Các tác vụ nặng hoặc logic cũ sẽ được đẩy qua cho một nhóm **Node.js Worker** luôn ở trạng thái "nóng" thông qua IPC (Inter-Process Communication).

### Phía Bun (Gateway)
```typescript
// Khởi động Node Worker thường trú
const nodeWorker = Bun.spawn(["node", "heavy-lifter.js"], {
  stdin: "pipe",
  stdout: "pipe",
});

Bun.serve({
  port: 3000,
  async fetch(req) {
    if (req.url.endsWith("/heavy")) {
      const data = await req.json();
      // Gửi task qua IPC
      nodeWorker.stdin.write(JSON.stringify(data) + "\n");
      // Đọc kết quả từ stdout (logic đơn giản hóa)
      const reader = nodeWorker.stdout.getReader();
      const { value } = await reader.read(); 
      return Response.json(JSON.parse(new TextDecoder().decode(value)));
    }
    return new Response("OK");
  },
});
```

---

## 2. Tối ưu I/O: Zero-Copy với Bun

Thay vì dùng `fs.readFile` truyền thống (gây copy dữ liệu nhiều lần giữa Kernel và User Space), sử dụng `Bun.file()` để đẩy thẳng dữ liệu từ ổ cứng ra card mạng.

```javascript
// Tốc độ tăng gấp 3 lần cho tài nguyên tĩnh
Bun.serve({
  fetch(req) {
    return new Response(Bun.file("./big-config.json"));
  }
});
```

---

## 3. Micro-batching: Xếp hàng xử lý

Để giảm tải CPU khi có hàng nghìn request cùng lúc, áp dụng một cửa sổ đệm (buffer window) cực nhỏ (khoảng 3ms) để gom các request đơn lẻ thành một mảng xử lý một lần.

```javascript
let buffer = [];
let timer = null;

function processBatch() {
  const currentBatch = buffer;
  buffer = [];
  timer = null;
  askNode({ type: 'batch', items: currentBatch });
}

function enqueue(item) {
  buffer.push(item);
  if (!timer) timer = setTimeout(processBatch, 3);
}
```

---

## 4. Quản lý bộ nhớ & Garbage Collection

Tránh khởi tạo Object mới (`new DatabaseClient()`, `new RegExp()`) bên trong vòng lặp xử lý request. 
- **Giải pháp:** Đưa tất cả các instance tái sử dụng được ra phạm vi toàn cục (**Global Scope**). Điều này giúp giảm áp lực lên Garbage Collection (GC) và tránh làm nổ tung bộ nhớ khi traffic cao.

---

## 5. Chiến lược Cache 2 lớp

Giảm phụ thuộc vào kết nối TCP tới Redis bằng cách tận dụng tốc độ đọc file của Bun:
1. **L1 (Memory Cache):** Lưu 1000 key nóng nhất bằng LRU (phản hồi tính bằng micro giây).
2. **L2 (File Cache):** Ghi dữ liệu ít nóng hơn ra file JSON trong `/tmp/cache/`.

---

## 6. Native API vs NPM Bloat

Loại bỏ các package NPM không cần thiết để giảm I/O overhead và rút ngắn thời gian Cold Start.
- Thay `uuid` bằng `crypto.randomUUID()`.
- Thay `qs` bằng `URLSearchParams`.
- Tận dụng Native API đã được tối ưu bằng C++ trong Bun/Node hiện đại.

---

## 7. Quản lý môi trường phát triển

Trong kiến trúc lai, việc quản lý nhiều phiên bản runtime (Node cũ cho legacy, Bun mới cho gateway) thường gây xung đột.
- **Công cụ đề xuất:** **ServBay**.
- **Lợi ích:** Chạy song song đa phiên bản Node (14, 22) và Bun 1.1 trong môi trường cách ly hoàn chỉnh, tích hợp sẵn DB (Redis, PostgreSQL) chỉ với một cú click.

---

## 🏁 Lời kết

Để đạt được độ trễ **10ms**, chìa khóa không nằm ở việc chọn một công cụ duy nhất, mà là sự phối hợp:
- **Bun:** Tốc độ I/O và Gateway.
- **Node.js:** Xử lý logic nghiệp vụ ổn định.
- **Kiến trúc:** Bất đồng bộ, Micro-batching và Zero-copy.

---
*Tóm tắt từ bài viết của **Lamri Abdellah Ramdane** trên Viblo.*