Dưới đây là nội dung được chuyển đổi thành một tệp `README.md` chuyên nghiệp, cấu trúc rõ ràng và dễ theo dõi.

---

# Change Data Capture (CDC) - Thu thập Thay đổi Dữ liệu Thời gian thực

## 📌 Tổng quan
Trong các hệ thống hiện đại, việc đồng bộ dữ liệu giữa cơ sở dữ liệu (DB) chính với các hệ thống hạ nguồn (Data Warehouse, Elasticsearch, Redis) là một thách thức lớn. Phương pháp sao chép hàng loạt (Batch Processing) truyền thống thường gây quá tải DB và có độ trễ cao.

**Change Data Capture (CDC)** là giải pháp tối ưu giúp theo dõi, nắm bắt và truyền tải mọi thay đổi dữ liệu (Insert, Update, Delete) ngay khi chúng xảy ra theo thời gian thực.

---

## 📖 Mục lục
1. [CDC là gì?](#cdc-là-gì)
2. [Cách hoạt động](#cách-hoạt-động)
3. [Tại sao cần CDC?](#tại-sao-cần-cdc)
4. [Các phương pháp triển khai](#các-phương-pháp-triển-khai)
5. [Công cụ phổ biến](#công-cụ-phổ-biến)
6. [Trường hợp sử dụng thực tế](#trường-hợp-sử-dụng-thực-tế)

---

## 🔍 CDC là gì?
CDC là kỹ thuật nhận diện và theo dõi các thay đổi gia tăng trên dữ liệu. Thay vì sao chép toàn bộ bảng định kỳ, CDC chỉ tập trung vào:
*   **INSERT**: Thêm dữ liệu mới.
*   **UPDATE**: Cập nhật dữ liệu hiện có.
*   **DELETE**: Xóa dữ liệu.

---

## ⚙️ Cách hoạt động

### Quy trình 3 bước:
1.  **Sự kiện nguồn**: Có thay đổi trong DB (Ví dụ: Đơn hàng mới được tạo).
2.  **Nắm bắt thay đổi**: CDC ghi lại loại thao tác, dữ liệu trước/sau thay đổi, timestamp và mã giao dịch.
3.  **Phân phối**: Gửi dữ liệu đến các hệ thống hạ nguồn (Kafka, BigQuery, Elasticsearch, Redis...).

### Ví dụ về một bản tin CDC (JSON):
```json
{
  "operation": "UPDATE",
  "table": "orders",
  "timestamp": "2024-12-26 10:05:30",
  "before": { "id": 1, "status": "pending", "total": 100.00 },
  "after": { "id": 1, "status": "shipped", "total": 100.00 },
  "transaction_id": "abc123"
}
```

---

## 🚀 Tại sao cần CDC?

| Đặc điểm | Sao chép hàng loạt (Batch) | Change Data Capture (CDC) |
| :--- | :--- | :--- |
| **Tải lên DB** | Rất nặng (Quét toàn bộ bảng) | Rất nhẹ (Chỉ đọc thay đổi) |
| **Độ trễ** | Cao (vài giờ đến 24h) | Thấp (mili giây đến vài giây) |
| **Dữ liệu Xóa** | Khó theo dõi | Theo dõi chính xác |
| **Lịch sử** | Chỉ thấy trạng thái cuối cùng | Thấy toàn bộ quá trình thay đổi |

### Lợi ích chính:
*   **Giảm tải hệ thống:** Không cần chạy các câu lệnh SQL SELECT lớn vào ban đêm.
*   **Đồng bộ thời gian thực:** Phục vụ phát hiện gian lận và báo cáo tức thì.
*   **Kiến trúc hướng sự kiện (Event-driven):** Biến DB thành một nguồn phát sự kiện cho các Microservices.

---

## 🛠 Các phương pháp triển khai

### 1. Dựa trên Nhật ký giao dịch (Log-based) - **Khuyến nghị**
Đọc các tệp nhật ký nội bộ của DB (MySQL Binlog, Postgres WAL).
*   **Ưu điểm:** Hiệu suất cao nhất, không ảnh hưởng đến ứng dụng, bắt được lệnh DELETE.
*   **Công cụ:** Debezium, Maxwell.

### 2. Dựa trên Dấu thời gian (Timestamp-based)
Truy vấn các hàng có cột `updated_at` mới.
*   **Ưu điểm:** Dễ triển khai.
*   **Nhược điểm:** Không bắt được dữ liệu đã bị xóa cứng (Hard Delete).

### 3. Dựa trên Kích hoạt (Trigger-based)
Sử dụng Trigger của DB để ghi thay đổi vào một bảng phụ.
*   **Ưu điểm:** Đáng tin cậy trong phạm vi DB.
*   **Nhược điểm:** Làm chậm tốc độ ghi của ứng dụng chính.

---

## 🧰 Công cụ phổ biến

*   **Debezium:** Nền tảng mã nguồn mở hàng đầu dựa trên Kafka Connect. Hỗ trợ MySQL, Postgres, MongoDB, SQL Server.
*   **AWS DMS:** Dịch vụ quản lý của Amazon giúp di chuyển dữ liệu liên tục.
*   **Goldengate:** Giải pháp cao cấp của Oracle dành cho doanh nghiệp lớn.
*   **Fivetran/Airbyte:** Các công cụ ELT hiện đại hỗ trợ CDC tích hợp.

---

## 💡 Trường hợp sử dụng thực tế

### 1. Thương mại điện tử
*   Đồng bộ tồn kho từ DB sang **Redis** để truy xuất nhanh.
*   Cập nhật thông tin sản phẩm sang **Elasticsearch** để tìm kiếm tức thì.

### 2. Vô hiệu hóa bộ nhớ đệm (Cache Invalidation)
Khi dữ liệu trong DB thay đổi, CDC phát sự kiện để xóa/cập nhật khóa tương ứng trong **Redis/Memcached**, đảm bảo người dùng không thấy dữ liệu cũ.

### 3. Tích hợp Microservices
Giúp các dịch vụ độc lập giao tiếp với nhau qua **Kafka**. Ví dụ: Dịch vụ Đơn hàng cập nhật DB -> CDC bắn sự kiện -> Dịch vụ Giao hàng nhận tin và tạo nhãn vận chuyển.

### 4. Phân tích dữ liệu (Real-time Analytics)
Đẩy dữ liệu liên tục về **Snowflake** hoặc **BigQuery** để bộ phận kinh doanh có báo cáo mới nhất sau mỗi giây.

---
*Tài liệu này được tổng hợp để cung cấp cái nhìn tổng quan về công nghệ CDC trong kiến trúc dữ liệu hiện đại.*