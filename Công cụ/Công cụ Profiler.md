Dưới đây là nội dung bài viết được chuyển đổi sang định dạng **README.md** để bạn có thể sử dụng cho các kho lưu trữ mã nguồn (GitHub/GitLab) hoặc tài liệu kỹ thuật của mình.

---

# 🚀 5 "Invisible" Bottlenecks That Profilers Never Show You

[![Author: Nguyễn Huy Hoàng](https://img.shields.io/badge/Author-Nguy%E1%BB%85n%20Huy%20Ho%C3%A0ng-blue)](https://viblo.asia/u/hhoang)
[![Category: Backend](https://img.shields.io/badge/Category-Backend-green)](#)
[![Performance: Optimization](https://img.shields.io/badge/Performance-Optimization-orange)](#)

Đã bao giờ bạn rơi vào tình huống: Người dùng than phiền API chậm, tỷ lệ timeout tăng cao, nhưng Dashboard quản trị vẫn cho thấy mọi chỉ số đều "đẹp như mơ"? CPU 10-20%, RAM dư dả, Disk I/O thấp.

Nếu API của bạn đôi lúc chậm nhưng tài nguyên hệ thống vẫn thấp, rất có thể bottleneck đang nằm ở những "điểm mù" mà các công cụ Profiler thông thường không bao giờ quét tới.

---

## 📌 Mục lục
1. [DNS Lookup – Kẻ ngáng đường ở cổng chào](#1-dns-lookup--kẻ-ngáng-đường-ở-cổng-chào)
2. [TLS Handshake – Cái bắt tay quá rườm rà](#2-tls-handshake--cái-bắt-tay-quá-rườm-rà)
3. [Connection Pool Lock – Xếp hàng trong im lặng](#3-connection-pool-lock--xếp-hàng-trong-im-lặng)
4. [Thread Starvation – Khi "thợ xây" ngồi chơi xào thịt](#4-thread-starvation--khi-thợ-xây-ngồi-chơi-xào-thịt)
5. [GC Pause (Stop-the-world) – Những khoảng lặng chết chóc](#5-gc-pause-stop-the-world--những-khoảng-lặng-chết-chóc)

---

## 1. DNS Lookup – Kẻ ngáng đường ở cổng chào
Mỗi lần thực hiện gọi một external API (SMS Gateway, Payment, Microservices), hệ thống có thể phải thực hiện truy vấn DNS.

*   **Vấn đề:** Nếu cấu hình `/etc/resolv.conf` không tối ưu hoặc DNS server của ISP/Cloud chậm, mỗi request mất thêm **50ms - 500ms** chỉ để tìm địa chỉ IP.
*   **Tại sao Profiler không thấy:** Profiler đo thời gian thực thi mã nguồn. DNS lookup nằm ở tầng OS/Network, diễn ra trước khi kết nối TCP được thiết lập.
*   **Giải pháp:**
    *   Sử dụng DNS Caching ở tầng ứng dụng hoặc OS (ví dụ: `nscd`).
    *   Với Go: Tùy chỉnh `Dialer` để giữ kết nối lâu hơn.

## 2. TLS Handshake – Cái bắt tay quá rườm rà
Bảo mật HTTPS luôn đi kèm với chi phí về hiệu năng.

*   **Vấn đề:** Để thiết lập TLS, client và server phải trao đổi qua lại nhiều lượt (Round-trips). Nếu latency giữa 2 server là 50ms, việc "bắt tay" có thể tốn **150-200ms** trước khi truyền dữ liệu.
*   **Dấu hiệu:** Latency cao khi gọi service ngoài, nhưng kiểm tra trực tiếp bằng `curl` từ server thì kết quả không ổn định.
*   **Giải pháp:**
    *   Sử dụng **Keep-Alive** để tái sử dụng kết nối TCP/TLS.
    *   Tuyệt đối không khởi tạo HTTP Client mới cho mỗi request. Hãy dùng một **Shared Client** với Connection Pool được cấu hình tốt.

## 3. Connection Pool Lock – Xếp hàng trong im lặng
Đây là vấn đề phổ biến với Database (SQL) hoặc Redis.

*   **Vấn đề:** Cấu hình `MaxOpenConns` quá thấp. Khi hết kết nối khả dụng, các request mới phải đứng đợi (queue) để chiếm lock của pool.
*   **Tại sao khó tìm:** Profiler báo hàm `QueryContext` chạy rất nhanh (thời gian tại DB thấp), nhưng không tính thời gian request phải "xếp hàng" chờ lấy connection.
*   **Giải pháp:**
    *   Monitor chỉ số `WaitDuration` của connection pool.
    *   Nới rộng Pool hoặc tối ưu hóa lại câu lệnh query.

## 4. Thread Starvation – Khi "thợ xây" ngồi chơi xào thịt
Phổ biến trong Node.js (Event Loop) hoặc Worker Pool trong Go/Java.

*   **Vấn đề:** Một tác vụ nặng về tính toán (CPU bound) hoặc code đồng bộ (Sync) chặn đứng luồng xử lý chính. Trong Go, lạm dụng `runtime.LockOSThread()` hoặc quá nhiều CGO call không kiểm soát có thể gây nghẽn Scheduler.
*   **Hệ quả:** Hệ thống không thực sự "bận" (CPU thấp), nhưng request mới không được tiếp nhận vì không còn thread nào rảnh.
*   **Giải pháp:**
    *   Tách biệt tác vụ nặng ra worker pool riêng.
    *   Tránh block Event Loop bằng mọi giá.

## 5. GC Pause (Stop-the-world) – Những khoảng lặng chết chóc
Dù GC trong Go đã được tối ưu xuống dưới 1ms, nhưng với hệ thống tải cao, nó vẫn là vấn đề lớn.

*   **Vấn đề:** Tạo quá nhiều object tạm thời (struct, string concat liên tục) khiến GC phải quét liên tục, gây ra hiện tượng "Stuttering" (khựng hệ thống).
*   **Tại sao khó bắt:** Các công cụ Monitoring theo giây (1s, 5s) thường lấy số trung bình (average), làm mượt các "spike" latency khiến bạn không thấy được sự sụt giảm ở mức milisecond.
*   **Giải pháp:**
    *   Sử dụng `sync.Pool` để tái sử dụng object.
    *   Hạn chế ép kiểu (`interface{}`) hoặc cấp phát bộ nhớ trong vòng lặp.

---

## 💡 Lời kết
Để tối ưu Backend, đừng chỉ nhìn vào Profiler. Hãy mở rộng góc nhìn ra tầng **Hệ điều hành, Mạng và Cơ chế quản lý tài nguyên**.

> Khi hệ thống chậm mà CPU vẫn thấp: Đừng vội Scale up. Hãy kiểm tra DNS, Connection Pool và TLS.

---
*Nội dung được tóm tắt từ bài chia sẻ của tác giả **Nguyễn Huy Hoàng** trên Viblo.*

**Tag:** #Backend #Performance #DevOps #SystemDesign #Go #NodeJS #Optimization