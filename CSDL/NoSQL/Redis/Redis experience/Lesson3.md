Dưới đây là nội dung bài viết được chuyển đổi thành tệp **README.md** chuyên nghiệp, tóm tắt các kịch bản lỗi và giải pháp tối ưu cho hệ thống Caching sử dụng Redis.

---

# 🛡️ Redis Caching: Từ "Tấm Khiên" đến "Tử Huyệt" & Chiến Lược Phòng Thủ

![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![System Design](https://img.shields.io/badge/System-Design-orange?style=for-the-badge)
![High Availability](https://img.shields.io/badge/High-Availability-green?style=for-the-badge)

Tài liệu này phân tích 3 thảm họa kinh điển khi sử dụng Redis Cache trong các hệ thống lớn (High Traffic) và các giải pháp chuẩn Enterprise để bảo vệ Database.

---

## 📖 1. Mô hình Cache-Aside (Tiêu chuẩn)

Hầu hết các hệ thống hiện đại sử dụng mẫu thiết kế **Cache-Aside** với luồng xử lý:
1. **Kiểm tra Cache:** Nếu dữ liệu có trong Redis (**Cache Hit**) -> Trả về ngay (Latency ~1ms).
2. **Truy vấn DB:** Nếu không có (**Cache Miss**) -> Truy vấn Database (Latency ~50ms).
3. **Cập nhật Cache:** Trả kết quả cho User và đồng thời lưu bản sao vào Redis cho các yêu cầu sau.

---

## 🚨 2. Ba "Kỵ Sĩ Khải Huyền" của Hệ thống Cache

### Thảm họa 1: Cache Penetration (Xuyên thủng Cache)
**Kịch bản:** Hacker liên tục gọi API với các ID không tồn tại (VD: ID = -1, ID = 999999).
- **Vấn đề:** Vì ID không tồn tại, Redis luôn báo *Cache Miss*. App liên tục truy vấn xuống Database. Database không tìm thấy dữ liệu (NULL) nên App không lưu vào Cache.
- **Hệ quả:** Hàng vạn request đập trực tiếp vào Database, làm sập hệ thống.

**Giải pháp:**
- **Cache Null:** Lưu luôn giá trị NULL vào Redis với TTL ngắn (VD: 30s).
- **Bloom Filter:** Sử dụng lưới lọc xác suất để kiểm tra sự tồn tại của ID trước khi truy vấn Cache/DB.

---

### Thảm họa 2: Cache Breakdown (Đánh thủng Cache - Hot Key)
**Kịch bản:** Một nội dung cực kỳ "nóng" (Hot Key) đang có hàng trăm nghìn lượt truy cập mỗi giây bỗng dưng hết hạn (TTL = 0).
- **Vấn đề:** Ngay mili-giây Key biến mất, toàn bộ request đồng loạt nhận *Cache Miss* và cùng lúc ùa xuống Database để lấy dữ liệu mới.
- **Hệ quả:** Database bị đột quỵ do quá tải kết nối đột ngột.

**Giải pháp:**
- **Mutex Lock (Khóa phân tán):** Chỉ cho phép request đầu tiên chiếm được khóa (Sử dụng `SETNX`) xuống DB. Các request khác phải đợi và thử lại sau khi Cache đã được cập nhật.

```javascript
// Minh họa logic Mutex Lock (Node.js)
async function getArticle(id) {
    let article = await redis.get(`article:${id}`);
    if (article) return JSON.parse(article);

    // Cố gắng giành khóa (lock) trong 5 giây
    const lock = await redis.set(`lock:article:${id}`, "locked", "NX", "EX", 5);
    
    if (lock) {
        article = await db.query(`SELECT * FROM articles WHERE id = ?`, [id]);
        await redis.set(`article:${id}`, JSON.stringify(article), "EX", 3600);
        await redis.del(`lock:article:${id}`);
        return article;
    } else {
        await sleep(50); // Đợi 50ms và thử lại
        return getArticle(id);
    }
}
```

---

### Thảm họa 3: Cache Avalanche (Tuyết lở Cache)
**Kịch bản:** Một lượng lớn Key được thiết lập thời gian hết hạn (TTL) giống hệt nhau hoặc cùng lúc Redis Server gặp sự cố.
- **Vấn đề:** Toàn bộ Key đồng loạt biến mất trong cùng một thời điểm.
- **Hệ quả:** Database hứng trọn "cơn lũ" request cập nhật lại dữ liệu cho hàng vạn Key khác nhau.

**Giải pháp:**
- **TTL Jitter:** Không bao giờ đặt TTL cố định. Hãy cộng thêm một khoảng thời gian ngẫu nhiên.
  - *VD:* `TTL = 3600 + random(0, 300)`.
- **Hạ tầng:** Sử dụng Redis Cluster hoặc Sentinel để đảm bảo tính sẵn sàng cao.

---

## 💡 3. Chuyên sâu: Sliding Expiration vs. Consistency

Một kỹ thuật phổ biến khác là **Sliding Expiration** (Gia hạn trượt - cứ đọc là gia hạn TTL).

| Đặc điểm | Nội dung (Sản phẩm/Bài viết) | Phiên đăng nhập (User Session) |
| :--- | :--- | :--- |
| **Sử dụng** | **Không khuyến khích** | **Tiêu chuẩn vàng** |
| **Lý do** | Gây ra tình trạng "Dữ liệu ôi thiu" (Stale Data). Admin sửa giá nhưng User liên tục F5 khiến Key không bao giờ chết để cập nhật mới. | Nếu User còn hoạt động (còn thở) thì phiên đăng nhập còn tồn tại. |
| **Hiệu năng** | Tốn chi phí ghi (Write Amplification) do phải cập nhật TTL liên tục. | Chấp nhận được vì tính chất đặc thù của Session. |

---

## 🏁 Lời kết

Thiết kế hệ thống Cache không chỉ là lưu trữ dữ liệu vào RAM, mà là nghệ thuật cân bằng giữa **Độ trễ (Latency)** và **Độ chính xác (Consistency)**. Hãy luôn trang bị các lớp phòng thủ như Mutex Lock và TTL Jitter để bảo vệ "trái tim" Database của bạn.

---
*Tài liệu được tổng hợp dựa trên series kiến thức của Nguyễn Huy Hoàng (Software Engineer).*