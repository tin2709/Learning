Dưới đây là nội dung bài viết được chuyển đổi thành định dạng **README.md** chuyên nghiệp, phù hợp để lưu trữ trong kho mã nguồn hoặc tài liệu kỹ thuật của bạn.

---

# 🚀 Redis Mastery: Khai Mở Sức Mạnh 5 Cấu Trúc Dữ Liệu Cốt Lõi

![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![Backend](https://img.shields.io/badge/Backend-Development-blue?style=for-the-badge)

Đừng biến Redis thành một "bãi rác" chỉ chứa các chuỗi JSON khổng lồ. Tài liệu này hướng dẫn cách vứt bỏ tư duy `JSON.stringify()` để tận dụng tối đa các cấu trúc dữ liệu nguyên bản của Redis, giúp tối ưu hiệu năng, băng thông và CPU.

---

## 📌 Vấn đề: Thảm họa "JSON.stringify"

Nhiều lập trình viên có thói quen đóng gói toàn bộ Object thành chuỗi JSON và dùng lệnh `SET`/`GET`. 

**Ví dụ bài toán Giỏ hàng (Cart):**
1. **Cách làm cũ:** `GET` toàn bộ JSON -> `JSON.parse` -> Cập nhật số lượng -> `JSON.stringify` -> `SET` ngược lại.
2. **Hệ quả:** 
   - Lãng phí CPU và băng thông mạng.
   - Dễ xảy ra lỗi **Race Condition** (xung đột dữ liệu khi nhiều người cùng thao tác).

**Giải pháp:** Sử dụng 5 vũ khí tối thượng dưới đây của Redis.

---

## 🛠 5 Cấu Trúc Dữ Liệu Tối Thượng

### 1. Strings (Chuỗi) - Kẻ Đếm Số Nguyên Tử
Redis hiểu các giá trị số bên trong String và cho phép thực hiện toán tử trực tiếp trên RAM.

*   **Tính năng chính:** `INCR` (Increment).
*   **Ưu điểm:** Thao tác **Atomic** (Nguyên tử) - Đảm bảo tính chính xác tuyệt đối ngay cả khi có hàng vạn truy cập cùng lúc.
*   **Ứng dụng:** Đếm lượt xem bài viết, lượt click quảng cáo.
*   **Lệnh ví dụ:**
    ```bash
    INCR views:post_100
    ```

### 2. Hashes (Bảng Băm) - Cứu Tinh Của Object
Lưu trữ dữ liệu dưới dạng các trường (field) và giá trị (value) bên trong một Key, tương tự như một Object trong JavaScript.

*   **Tính năng chính:** `HSET`, `HINCRBY`.
*   **Ưu điểm:** Cập nhật từng thuộc tính nhỏ mà không cần lôi cả Object lớn về.
*   **Ứng dụng:** Thông tin User, chi tiết giỏ hàng (Cart).
*   **Lệnh ví dụ:**
    ```bash
    # Lưu giỏ hàng
    HSET cart:user_123 item1 2 item2 5
    # Tăng số lượng item1 thêm 1
    HINCRBY cart:user_123 item1 1
    ```

### 3. Lists (Danh sách liên kết) - Trùm Làm Hàng Đợi
Bản chất là **Doubly Linked List**. Việc chèn vào đầu hoặc cuối là cực nhanh O(1).

*   **Tính năng chính:** `LPUSH`, `LTRIM`.
*   **Ưu điểm:** Duy trì thứ tự dữ liệu theo thời gian thực.
*   **Ứng dụng:** Message Queue, Lịch sử hoạt động (Activity Stream), Log hệ thống.
*   **Lệnh ví dụ:**
    ```bash
    # Thêm log mới
    LPUSH user:123:logs "Login at 9:00 AM"
    # Giữ lại 50 log mới nhất và xóa phần cũ
    LTRIM user:123:logs 0 49
    ```

### 4. Sets (Tập hợp vô hướng) - Vua Lọc Trùng Lặp
Chứa các phần tử không có thứ tự và đặc biệt là **duy nhất** (không trùng lặp).

*   **Tính năng chính:** `SADD`, `SCARD`.
*   **Ưu điểm:** Chống spam, tự động loại bỏ các phần tử trùng lặp.
*   **Ứng dụng:** Tính năng Like (mỗi user chỉ like 1 lần), danh sách IP đen (Blacklist).
*   **Lệnh ví dụ:**
    ```bash
    # Thêm user 99 vào danh sách like post 1
    SADD post:1:likes 99
    # Đếm tổng số like
    SCARD post:1:likes
    ```

### 5. Sorted Sets (ZSET) - Trùm Cuối Bảng Xếp Hạng
Tương tự như Set nhưng mỗi phần tử đi kèm một điểm số (**Score**). Redis tự động sắp xếp các phần tử dựa trên điểm số này.

*   **Tính năng chính:** `ZADD`, `ZREVRANGE`.
*   **Ưu điểm:** Không cần chạy `ORDER BY` ở Database, dữ liệu luôn được sắp xếp sẵn trên RAM.
*   **Ứng dụng:** Bảng xếp hạng Game (Leaderboard), Top Trending, Phân tích dữ liệu theo thời gian.
*   **Lệnh ví dụ:**
    ```bash
    # Thêm điểm cho người chơi
    ZADD game:leaderboard 800 "Hoàng1"
    ZADD game:leaderboard 500 "HOàng"
    # Lấy Top 10 người cao điểm nhất
    ZREVRANGE game:leaderboard 0 9 WITHSCORES
    ```

---

## 💡 Kết Luận

Sử dụng đúng cấu trúc dữ liệu không chỉ giúp hệ thống chạy nhanh hơn mà còn giúp code của bạn trở nên **thanh lịch và ít lỗi hơn**. 

> "Đừng dùng Ferrari chỉ để đi chợ mua rau. Đừng dùng Redis chỉ để lưu chuỗi JSON."

---

## 📅 Roadmap Tiếp Theo
- [ ] Vũ khí chống Spam: **Rate Limiting bằng Redis (Sliding Window)**.
- [ ] Kỹ thuật nâng cao: **Redlock & Distributed Locks**.

---

## 👤 Thông tin Tác giả
*   **Tác giả:** Nguyễn Huy Hoàng (Software Engineer)
*   **Email:** hhoang02052004@gmail.com
*   **GitHub:** [HuyHoangCoder](https://github.com/HuyHoangCoder)
*   **Phone:** 0941 280 073

---
*Dựa trên bài viết gốc của Nguyễn Huy Hoàng tại Viblo.*