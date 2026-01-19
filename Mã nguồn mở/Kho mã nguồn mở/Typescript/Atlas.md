Dưới đây là bản phân tích chi tiết về dự án **Atlas CMMS** dựa trên cấu trúc mã nguồn và các file cấu hình bạn đã cung cấp.

---

# 📝 Phân tích Dự án Atlas CMMS

**Atlas CMMS** là một hệ thống quản lý bảo trì tập trung (Computerized Maintenance Management System), được ví như "Jira dành cho kỹ thuật viên". Hệ thống cho phép quản lý tài sản (Assets), lệnh làm việc (Work Orders), bảo trì phòng ngừa (Preventive Maintenance) và kho bãi (Inventory).

---

## 1. 🛠 Công nghệ cốt lõi (Core Tech Stack)

Dự án sử dụng mô hình Monorepo (hoặc tập trung mã nguồn) với các công nghệ hiện đại:

*   **Backend:** Java 17, Spring Boot 3.2.x.
    *   **Data Access:** Spring Data JPA, Hibernate Envers (truy vết lịch sử thay đổi).
    *   **Database Migration:** Liquibase (quản lý phiên bản DB).
    *   **Security:** Spring Security, JWT (JSON Web Token), OAuth2 (Google, Microsoft SSO).
    *   **Job Scheduling:** Quartz Scheduler (cho các tác vụ lặp lại như tạo lệnh bảo trì định kỳ).
    *   **Storage:** Hỗ trợ đa dạng giữa MinIO (Local) và Google Cloud Storage (GCP).
    *   **API Documentation:** SpringDoc / OpenAPI (Swagger).
*   **Frontend:** React, TypeScript, Material UI (MUI).
    *   **State Management:** Redux Toolkit.
    *   **Styling:** Emotion/Styled-components.
*   **Mobile:** React Native (Expo), TypeScript.
*   **Infrastructure:** Docker, Docker Compose, PostgreSQL 16.

---

## 2. 🏗 Tư duy kiến trúc (Architectural Thinking)

Hệ thống được thiết kế với các tư duy kiến trúc bài bản:

*   **Kiến trúc Đa người thuê (Multi-tenancy):** Thông qua `TenantAspect.java` và các thực thể kế thừa `CompanyAudit`, hệ thống đảm bảo dữ liệu của các công ty (Tenant) khác nhau hoàn toàn cô lập. Dữ liệu được lọc theo `company_id` ở mức toàn cục hoặc qua Aspect/Interceptor.
*   **Phân quyền dựa trên vai trò (RBAC - Role Based Access Control):** Không chỉ dừng lại ở Role, hệ thống phân tách chi tiết các quyền (Permissions) như: `VIEW`, `CREATE`, `EDIT_OTHER`, `DELETE_OTHER` cho từng thực thể (Asset, Work Order, Part...).
*   **Phân tách Logic (Decoupling):** Sử dụng mô hình chuẩn: `Controller` -> `Service` -> `Repository` -> `Model`. Sử dụng `MapStruct` để chuyển đổi qua lại giữa `Entity` và `DTO`, giúp bảo vệ cấu trúc DB bên trong.
*   **Khả năng mở rộng (Extensibility):** Hỗ trợ `Custom Fields` (trường tùy chỉnh), cho phép người dùng thêm các thông tin đặc thù cho tài sản hoặc lệnh làm việc mà không cần thay đổi code.

---

## 3. 🚀 Các kỹ thuật then chốt (Key Techniques)

*   **Advanced Search & Filtering:** Lớp `SpecificationBuilder` và `WrapperSpecification` cho phép tạo các truy vấn động phức tạp từ Frontend (tìm kiếm theo nhiều điều kiện, toán tử AND/OR, LIKE, EQUAL...) mà không cần viết nhiều phương thức trong Repository.
*   **Auditing & History:** Sử dụng `Hibernate Envers` để tự động ghi lại mọi thay đổi của Lệnh làm việc (`WorkOrderHistory`). Điều này rất quan trọng trong bảo trì để biết ai đã sửa gì và khi nào.
*   **Quản lý tài liệu đa phương tiện:** Tích hợp tính năng ký số (Signature), ghi âm (Audio Description) và tải lên file. Hệ thống sử dụng `StorageServiceFactory` để chuyển đổi linh hoạt giữa lưu trữ nội bộ (MinIO) và đám mây (GCP).
*   **Bảo trì phòng ngừa (PM):** Sử dụng `Quartz` để quét các lịch trình bảo trì. Nếu đến hạn (theo thời gian hoặc theo chỉ số đồng hồ đo - Meter), hệ thống tự động sinh ra một `WorkOrder` mới.
*   **Quốc tế hóa (i18n):** Hỗ trợ hơn 14 ngôn ngữ thông qua các file `messages.properties` ở backend và cấu hình i18n ở frontend/mobile.

---

## 4. 🔄 Tóm tắt luồng hoạt động (Operational Flow)

### A. Luồng Đăng ký/Đăng nhập:
1.  Người dùng đăng ký -> Tạo `Company` mới -> Tạo `CompanySettings` và các `Role` mặc định.
2.  Đăng nhập -> Backend kiểm tra Credentials -> Trả về JWT Token chứa thông tin vai trò.

### B. Luồng Quản lý Tài sản (Asset Management):
1.  Người dùng tạo Asset (có thể gán vào Location, đính kèm tài liệu).
2.  Mỗi Asset có thể có các `Meter` (đồng hồ đo). Khi nhân viên nhập `Reading` (chỉ số), nếu chỉ số vượt ngưỡng, hệ thống sẽ tự kích hoạt Lệnh làm việc (Work Order).

### C. Luồng Lệnh làm việc (Work Order Lifecycle):
1.  **Khởi tạo:** User tạo Work Order (Yêu cầu sửa chữa hoặc Bảo trì định kỳ).
2.  **Phân công:** Giao cho kỹ thuật viên hoặc Team. Gửi thông báo (Push Notification qua Expo/WebSocket).
3.  **Thực hiện:** Kỹ thuật viên dùng Mobile app quét QR/NFC để mở Asset -> Bật Timer (`Labor`) để tính giờ làm -> Check các đầu mục công việc (`Checklist`).
4.  **Hoàn thành:** Chụp ảnh nghiệm thu, ký tên -> Hệ thống tính toán tổng chi phí (Phụ tùng + Nhân công + Chi phí ngoài).

### D. Luồng Báo cáo & Phân tích:
1.  Dữ liệu từ các bảng `WorkOrder`, `AssetDowntime`, `PartConsumption` được tổng hợp qua `AnalyticsController`.
2.  Frontend hiển thị các biểu đồ (Charts) về hiệu suất thiết bị (MTBF - Thời gian trung bình giữa các lần hỏng, MTTR - Thời gian sửa chữa trung bình).

---

## 5. 📂 Cấu trúc thư mục tóm tắt

*   **`api/`**: Mã nguồn Backend (Spring Boot).
    *   `src/main/java/com/grash/advancedsearch`: Logic tìm kiếm động.
    *   `src/main/resources/db/changelog`: Các file SQL/XML quản lý phiên bản DB.
*   **`frontend/`**: Mã nguồn Web (React).
    *   `src/content/own/Analytics`: Các màn hình báo cáo.
    *   `src/slices`: Quản lý trạng thái Redux.
*   **`mobile/`**: Mã nguồn App (React Native).
    *   `components/actionSheets`: Các menu tương tác nhanh trên mobile.
*   **`docker-compose.yml`**: File điều phối toàn bộ hệ thống (DB, API, Web, Storage).

**Kết luận:** Đây là một dự án có độ hoàn thiện cực kỳ cao, áp dụng đầy đủ các kỹ thuật lập trình doanh nghiệp (Enterprise Programming) và sẵn sàng cho việc triển khai thực tế ở quy mô lớn.