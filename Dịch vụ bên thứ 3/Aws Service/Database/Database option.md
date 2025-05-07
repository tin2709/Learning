# 1 Hướng dẫn Chọn Dịch vụ Cơ sở dữ liệu trên AWS: RDS, DynamoDB, và Aurora

Các dịch vụ cơ sở dữ liệu đóng vai trò cốt lõi trong hầu hết các ứng dụng và là kiến thức quan trọng cho kỳ thi AWS CCP (Certified Cloud Practitioner). Tài liệu này phân tích sâu về Amazon RDS, Amazon DynamoDB và Amazon Aurora để giúp bạn lựa chọn công cụ lưu trữ dữ liệu phù hợp nhất.

![alt text](image.png)

## 1. Amazon RDS - Relational Database Service

*   **Đặc điểm chính:**
    *   Dịch vụ cơ sở dữ liệu quan hệ (SQL) được quản lý hoàn toàn (Managed).
    *   Hỗ trợ nhiều công cụ database phổ biến: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, và cả Amazon Aurora.
    *   AWS chịu trách nhiệm các tác vụ quản trị như: cập nhật hệ điều hành (patch OS), sao lưu tự động (automatic backups), và khả năng chuyển đổi dự phòng Multi-AZ (Multi-AZ failover).

*   **Trường hợp sử dụng điển hình:**
    *   Các ứng dụng yêu cầu xử lý giao dịch (transactional) cao.
    *   Các ứng dụng với schema quan hệ chặt chẽ (relational schema), ví dụ: hệ thống thanh toán (billing system), hệ thống quản lý quan hệ khách hàng (CRM).

*   **Khả năng Mở rộng (Scaling) & Tính sẵn sàng cao (High Availability):**
    *   **Vertical Scaling:** Tăng kích thước instance (ví dụ: từ `db.t3` lên `db.m5`) để có thêm CPU, RAM. Yêu cầu downtime ngắn.
    *   **Read Replicas:** Tạo bản sao chỉ đọc để phân tải lưu lượng truy vấn đọc (offload read traffic). Hỗ trợ sao chép liên vùng (cross-Region).
    *   **Multi-AZ Deployment:** Tự động tạo bản sao dự phòng (standby copy) ở Availability Zone khác và tự động chuyển đổi (switchover) khi primary instance gặp sự cố, đảm bảo tính sẵn sàng cao.
    *   **Automated Backups & Snapshots:** Sao lưu tự động và tạo snapshots. Có thể lưu trữ tối đa 35 ngày. Hỗ trợ khôi phục tại một thời điểm cụ thể (point-in-time restore).

*   **Mô hình định giá:**
    *   **On-Demand Instance:** Thanh toán theo giờ sử dụng instance.
    *   **Reserved Instances:** Giảm giá 30–60% nếu cam kết sử dụng trong 1–3 năm.
    *   **Storage:** Tính phí theo dung lượng (USD/GB mỗi tháng) cộng với số lượng yêu cầu I/O (áp dụng cho Provisioned IOPS).
    *   **Data Transfer:** Miễn phí khi truyền dữ liệu trong cùng AZ (intra-AZ), tính phí khi truyền giữa các AZ (cross-AZ) và ra Internet.

## 2. Amazon DynamoDB - NoSQL Key‑Value & Document

*   **Đặc điểm chính:**
    *   Dịch vụ NoSQL (Key-Value & Document) không máy chủ (Serverless), được quản lý hoàn toàn (fully managed).
    *   Tự động mở rộng thông lượng (scale throughput) và dung lượng lưu trữ (storage).

*   **Mô hình dữ liệu:**
    *   Dữ liệu được tổ chức thành Bảng (Table), Mục (Item) và Thuộc tính (Attribute).
    *   Sử dụng Khóa chính (Primary Key) bao gồm Khóa phân vùng (Partition Key) và tùy chọn Khóa sắp xếp (Sort Key).

*   **Trường hợp sử dụng điển hình:**
    *   Lưu trữ thông tin phiên người dùng (session store).
    *   Giỏ hàng mua sắm (shopping cart).
    *   Bảng xếp hạng thời gian thực (real-time leaderboards).
    *   Các ứng dụng yêu cầu độ trễ thấp và khả năng mở rộng cực lớn.

*   **Khả năng (Capacity) & Tính năng:**
    *   **Chế độ Provisioned (Provisioned Mode):** Định trước số lượng Read Capacity Units (RCU) và Write Capacity Units (WCU). Cần dự báo trước nhu cầu.
    *   **Chế độ On-Demand (On-Demand Mode):** Tự động mở rộng quy mô, thanh toán theo số lượng yêu cầu đọc/ghi thực tế. Không cần dự báo trước.
    *   **Global Tables:** Cung cấp khả năng sao chép dữ liệu đa vùng (multi-Region replication) tự động.
    *   **Transactions & Streams:** Hỗ trợ các giao dịch ACID (trong một bảng hoặc giữa các bảng) và cung cấp luồng dữ liệu thay đổi (change data capture) với DynamoDB Streams.

*   **Mô hình định giá:**
    *   **Chế độ Provisioned:** Tính phí dựa trên số lượng RCU/WCU đã định trước (ví dụ: $0.00013 mỗi RCU/WCU mỗi tháng) cộng với phí lưu trữ ($0.25/GB mỗi tháng).
    *   **Chế độ On-Demand:** Tính phí dựa trên số lượng yêu cầu đọc/ghi thực tế (ví dụ: $1.25 cho mỗi triệu yêu cầu ghi + $0.25 cho mỗi triệu yêu cầu đọc).
    *   **Data Transfer & Streams:** Tính thêm phí dựa trên dung lượng truyền (GB mỗi tháng) và số lượng yêu cầu đối với Streams.

## 3. Amazon Aurora - High‑Performance Relational

*   **Đặc điểm chính:**
    *   Cơ sở dữ liệu quan hệ hiệu suất cao, tương thích với MySQL và PostgreSQL nhưng được AWS tối ưu riêng.
    *   Hiệu suất vượt trội: Nhanh hơn gấp 5 lần so với MySQL chuẩn và gấp 3 lần so với PostgreSQL chuẩn nhờ kiến trúc lưu trữ phân tán.
    *   Hỗ trợ Multi-Master (ghi đồng thời vào nhiều instance) và Serverless.

*   **Tính năng nâng cao:**
    *   **Aurora Serverless v2:** Tự động điều chỉnh năng lực tính toán (compute) theo nhu cầu, mở rộng linh hoạt từ mức tối thiểu đến tối đa chỉ trong mili giây.
    *   **Aurora Global Database:** Cung cấp khả năng sao chép dữ liệu đa vùng (cross-Region) với độ trễ thấp cho các ứng dụng toàn cầu.

*   **Khả năng Mở rộng (Scaling) & Định giá:**
    *   **Chế độ Provisioned:** Chọn lớp instance cụ thể. Có thể tạo tối đa 4 bản sao chỉ đọc (read replicas) trong cùng Region và tối đa 16 bản sao chỉ đọc cross-Region (với Aurora Global Database).
    *   **Chế độ Serverless v2:** Thanh toán theo Aurora Capacity Units (ACU) tiêu thụ. Khả năng tự động mở rộng nhanh chóng và chi tiết hơn Serverless v1.
    *   **Storage:** Tính phí theo dung lượng ($0.10/GB mỗi tháng) cộng với phí I/O dựa trên khối lượng công việc (workload).

## 4. So sánh và Hướng dẫn Lựa chọn

*   **Amazon RDS:** Lựa chọn phù hợp cho các cơ sở dữ liệu quan hệ thông thường, các ứng dụng giao dịch với khối lượng công việc có thể dự đoán được (predictable workload).
*   **Amazon DynamoDB:** Lý tưởng cho các ứng dụng cần khả năng mở rộng linh hoạt, mô hình dữ liệu không schema cố định (no-schema) hoặc schema linh hoạt, và yêu cầu độ trễ rất thấp. Thích hợp cho các workload có lưu lượng truy cập thay đổi đáng kể hoặc không thể dự báo.
*   **Amazon Aurora:** Dành cho các ứng dụng quan hệ yêu cầu hiệu suất cực cao và các tính năng nâng cao như Multi-Master, Serverless tự động mở rộng nhanh chóng hoặc cơ sở dữ liệu toàn cầu với độ trễ thấp. Thích hợp cho các doanh nghiệp lớn (enterprise) hoặc các ứng dụng quan trọng (mission-critical).

Việc lựa chọn dịch vụ database phù hợp trên AWS cần dựa trên việc đánh giá kỹ lưỡng tính chất dữ liệu, nhu cầu mở rộng (scale), yêu cầu về độ trễ (latency) và ngân sách chi phí của ứng dụng.