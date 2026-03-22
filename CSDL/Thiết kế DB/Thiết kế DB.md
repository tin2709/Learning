Dưới đây là nội dung bài viết về **Thiết kế Database cho Hệ thống Phân quyền IAM** được chuyển đổi sang định dạng **README.md** chuyên nghiệp.

---

# 🔐 Thiết kế Database Hệ thống Phân quyền IAM (Identity and Access Management)

[![Author: Hoang Minh Dai](https://img.shields.io/badge/Author-Hoang%20Minh%20Dai-blue)](https://viblo.asia/u/daihm-1041)
[![Architecture: RBAC](https://img.shields.io/badge/Architecture-RBAC%20%2B%20RLS-green)](#)
[![DB: PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-336791?logo=postgresql)](#)

Hệ thống IAM này được thiết kế theo mô hình **RBAC (Role-Based Access Control)** kết hợp với **Record-Level Security (RLS)**. Nó cho phép quản lý đa công ty (multi-tenant), kế thừa quyền và kiểm soát truy cập chi tiết đến từng bản ghi dựa trên điều kiện động.

---

## 📌 Mục lục
1. [Tổng quan hệ thống](#1-tổng-quan-hệ-thống)
2. [Chi tiết thực thể (Tables)](#2-chi-tiết-thực-thể-tables)
3. [Luồng kiểm tra quyền (Authorization Flow)](#3-luồng-kiểm-tra-quyền-authorization-flow)
4. [Trường hợp sử dụng thực tế](#4-trường-hợp-sử-dụng-thực-tế)
5. [Best Practices & Hiệu năng](#5-best-practices--hiệu-năng)
6. [Mở rộng nâng cao](#6-mở-rộng-nâng-cao)

---

## 1. Tổng quan hệ thống
Các thành phần cốt lõi:
*   **User & Company:** Quản lý đa công ty, một User có thể thuộc nhiều tổ chức.
*   **Group & Inheritance:** Nhóm quyền và cơ chế kế thừa (Group A bao gồm quyền Group B).
*   **Model-Level Access:** Phân quyền CRUD (Create, Read, Update, Delete) trên toàn bộ bảng.
*   **Record-Level Security:** Phân quyền chi tiết đến từng dòng dữ liệu bằng Domain Filter (JSON).

---

## 2. Chi tiết thực thể (Tables)

### 2.1. Quản lý Người dùng & Công ty
| Bảng | Mô tả | Key Columns |
| :--- | :--- | :--- |
| `IAM_USER` | Thông tin tài khoản & trạng thái | `id`, `email`, `company_id` (current) |
| `IAM_COMPANY` | Thông tin tổ chức/tenant | `id`, `name` |
| `IAM_USER_COMPANY` | Quan hệ N-N giữa User và Company | `user_id`, `company_id` |

### 2.2. Quản lý Nhóm & Kế thừa
*   **`IAM_GROUP`**: Định nghĩa vai trò (Admin, Manager, Employee...).
*   **`IAM_GROUP_IMPLIED`**: Cấu hình cây phân cấp quyền. Nếu Group A *implies* Group B, User trong Group A mặc nhiên có quyền của Group B.

### 2.3. Phân quyền Model (CRUD)
**`IAM_MODEL_ACCESS`**: Định nghĩa quyền cơ bản trên từng thực thể.
*   `perm_read`, `perm_write`, `perm_create`, `perm_unlink` (Boolean).

### 2.4. Phân quyền Bản ghi (Record-Level)
**`IAM_RECORD_RULE`**: Sử dụng **Domain Filter** để lọc dữ liệu động.
*   `domain` (JSONB): Lưu trữ điều kiện lọc. Ví dụ: `[["company_id", "=", "current_company_id"]]`.
*   `is_global`: Nếu true, áp dụng cho mọi user không phân biệt group.

---

## 3. Luồng kiểm tra quyền (Authorization Flow)

### 3.1. Kiểm tra Model-Level
1. Xác định danh sách **Effective Groups** (bao gồm các group trực tiếp và group được kế thừa).
2. Kiểm tra trong `IAM_MODEL_ACCESS`: Nếu **BẤT KỲ** group nào trong danh sách có quyền `perm_xxx = true` → Cho phép thực hiện thao tác.

### 3.2. Kiểm tra Record-Level
1. Sau khi vượt qua bước 3.1, lấy tất cả các `IAM_RECORD_RULE` tương ứng với model và effective groups.
2. Tổng hợp các domain filter bằng logic **OR**.
3. Inject điều kiện vào câu lệnh SQL (WHERE clause).

---

## 4. Trường hợp sử dụng thực tế

### Case 1: Multi-Tenant (Chỉ thấy dữ liệu công ty mình)
```json
// Domain filter áp dụng cho mọi bản ghi
[["company_id", "=", "current_company_id"]]
```

### Case 2: Kế thừa quyền (Hierarchy)
- **Admin** kế thừa **Manager**.
- **Manager** kế thừa **Employee**.
- Kết quả: Admin có toàn bộ quyền của Manager và Employee mà không cần cấu hình lại.

### Case 3: Quyền sở hữu (Chỉ thấy dữ liệu do mình tạo)
```json
[["salesperson_id", "=", "current_user_id"]]
```

---

## 5. Best Practices & Hiệu năng

### 🚀 Tối ưu hiệu năng
*   **Indexing:** Đánh index các khóa ngoại và cột hay lọc (`user_id`, `model_id`, `group_id`). Sử dụng **GIN index** cho cột `domain` nếu dùng JSONB.
*   **Caching:** Tính toán sẵn danh sách `Effective Groups` của User khi đăng nhập và lưu vào Redis/In-memory cache.

### 🛡️ Bảo mật
*   **Least Privilege:** Mặc định tất cả quyền là `false`. Chỉ cấp quyền khi cần thiết.
*   **Soft Delete:** Sử dụng cột `deleted_at` cho các bảng phân quyền để có thể truy vết và khôi phục khi cấu hình sai.

---

## 6. Mở rộng nâng cao
*   **Field-Level Security:** Kiểm soát quyền đọc/ghi trên từng cột (Column) của bảng.
*   **Time-Based Access:** Cấp quyền tạm thời có thời hạn (thêm cột `valid_from`, `valid_until`).
*   **Delegation:** Cho phép User ủy quyền (delegate) vai trò của mình cho người khác trong một khoảng thời gian.

---
*Tài liệu này dựa trên bài chia sẻ của **Hoang Minh Dai** trên nền tảng Viblo.*

**Tags:** #IAM #DatabaseDesign #RBAC #Security #PostgreSQL #BackendDevelopment