Dưới đây là nội dung bài viết được chuyển đổi thành tệp **README.md** chuyên nghiệp, cấu trúc rõ ràng và dễ dàng tra cứu cho các dự án kỹ thuật.

---

# 🛡️ Chiến Lược Chống Cache Stampede (Thundering Herd Problem)

![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![Database](https://img.shields.io/badge/Database-Security-blue?style=for-the-badge)
![High Availability](https://img.shields.io/badge/High-Availability-green?style=for-the-badge)

Tài liệu này phân tích về thảm họa **Cache Stampede** (Hiệu ứng bầy đàn) – một vấn đề phổ biến trong các hệ thống High Traffic – và các kỹ thuật phòng thủ sử dụng Redis để bảo vệ Database.

---

## 🌋 1. Hiện tượng Cache Stampede là gì?

**Cache Stampede** xảy ra khi một Key cực kỳ phổ biến (Hot Key) trong Cache hết hạn (TTL) đúng vào thời điểm có hàng nghìn request đồng thời.

### Kịch bản thảm họa:
1. **Giây thứ 0:** Key `top_products` hết hạn.
2. **Giây thứ 0.1:** 5.000 requests cùng lúc truy cập vào Redis và nhận kết quả **Cache Miss**.
3. **Giây thứ 0.2:** Cả 5.000 requests này cùng lúc "đâm" xuống Database để truy vấn dữ liệu.
4. **Hệ quả:** Database bị quá tải (CPU 100%), Connection Pool cạn kiệt, hệ thống treo hoặc sập hoàn toàn.

---

## 🛡️ 2. Kỹ thuật 1: Khóa phân tán (Distributed Lock / Mutex)

Đây là kỹ thuật "chặn cửa". Khi có hàng nghìn request cùng bị Cache Miss, hệ thống chỉ cho phép **duy nhất một request** xuống Database để cập nhật dữ liệu, các request còn lại phải chờ.

### Cơ chế hoạt động:
Sử dụng lệnh `SETNX` (Set if Not eXists) của Redis để tạo một ổ khóa tạm thời.

### Mã giả (Pseudo-code):
```javascript
async function getTopProducts() {
    let data = await redis.get("top_100");
    if (data) return data; // Cache Hit

    // Cache Miss -> Cố gắng giành "Khóa bảo vệ"
    const lockKey = "lock_top_100";
    const isLocked = await redis.set(lockKey, "locked", "NX", "EX", 10); 
    
    if (isLocked) {
        // Bạn là người duy nhất được xuống DB
        try {
            data = await db.query("SELECT ...");
            await redis.set("top_100", data, "EX", 300); // Cập nhật Cache
        } finally {
            await redis.del(lockKey); // Mở khóa
        }
        return data;
    } else {
        // Các request khác đứng chờ và thử lại
        await sleep(50);
        return getTopProducts(); // Đệ quy hoặc lặp lại
    }
}
```

---

## ⚡ 3. Kỹ thuật 2: Stale-While-Revalidate (Cho phép dữ liệu cũ)

Thay vì bắt người dùng chờ đợi, hệ thống chấp nhận trả về dữ liệu cũ đã hết hạn trong khi âm thầm cập nhật dữ liệu mới ở nền (background).

### Cơ chế hoạt động:
1. Không dùng cơ chế tự động xóa của Redis (`TTL`).
2. Nhúng một trường `expire_at` bên trong dữ liệu lưu trữ.
3. Khi nhận request:
   - Nếu `currentTime > expire_at`: Trả về dữ liệu hiện tại ngay lập tức, nhưng đồng thời kích hoạt một **Background Job** để cập nhật dữ liệu mới.
   - User không phải chờ đợi (Response time cực thấp).

---

## ⚙️ 4. Kỹ thuật 3: Background Warm-up (Gốc rễ nhàn nhã)

Tách biệt hoàn toàn luồng Đọc và Ghi. Đừng để API trực tiếp tham gia vào việc làm mới Cache.

### Cơ chế hoạt động:
- Xây dựng một **Worker/Cronjob** chạy ngầm.
- Ví dụ: Cứ mỗi 4 phút 50 giây, Worker tự động lấy dữ liệu từ DB và ghi đè (Overwrite) vào Redis.
- API người dùng chỉ có một nhiệm vụ duy nhất: **Đọc từ Redis**. Cache luôn luôn có dữ liệu, Database luôn được an toàn.

---

## 📊 So sánh các giải pháp

| Giải pháp | Độ phức tạp | Ưu điểm | Nhược điểm |
| :--- | :--- | :--- | :--- |
| **Distributed Lock** | Trung bình | Dữ liệu luôn mới nhất. | Một số request bị trễ (latency). |
| **Stale-While-Revalidate** | Cao | UX tuyệt vời, không độ trễ. | Chấp nhận dữ liệu cũ trong vài giây. |
| **Background Warm-up** | Thấp | DB cực kỳ an toàn, tách biệt logic. | Cần quản lý thêm các tác vụ chạy ngầm. |

---

## 💡 Kết luận

Database là thành phần mong manh nhất của hệ thống. Khi xử lý High Traffic, **code đúng thôi là chưa đủ**, bạn cần phải kiểm soát được dòng chảy của dữ liệu. Việc kết hợp giữa `TTL`, `Locking` và `Background Processing` chính là chìa khóa để xây dựng một hệ thống bền bỉ.

---

## 👤 Thông tin Tác giả
*   **Tác giả:** Nguyễn Huy Hoàng (Software Engineer)
*   **Email:** hhoang02052004@gmail.com
*   **GitHub:** [HuyHoangCoder](https://github.com/HuyHoangCoder)
*   **Phone:** 0941 280 073

---
*Nội dung được tổng hợp dựa trên series kinh nghiệm Redis của Nguyễn Huy Hoàng tại Viblo.*