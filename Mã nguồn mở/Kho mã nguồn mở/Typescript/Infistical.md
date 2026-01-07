Dưới đây là bản phân tích chi tiết về dự án **Infisical** dựa trên mã nguồn và cấu trúc thư mục bạn đã cung cấp, được trình bày dưới dạng một file `README_ANALYSIS.md` bằng tiếng Việt.

---

# Phân tích Hệ thống Quản lý Bí mật Infisical (Open-source)

Dự án Infisical là một nền tảng quản lý bí mật (secret management) hiện đại, được thiết kế để tập trung hóa cấu hình ứng dụng, thông tin xác thực, và quản lý hạ tầng PKI (Public Key Infrastructure).

## 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án được xây dựng theo mô hình Monorepo với các công nghệ chính sau:

### Backend (Node.js & TypeScript)
*   **Runtime:** Node.js (v20+).
*   **Framework:** **Fastify** - Một framework nhanh và ít tốn tài nguyên, sử dụng hệ thống Plugin mạnh mẽ.
*   **Database:** **PostgreSQL** kết hợp với **Knex.js** (Query Builder). Hệ thống sử dụng cơ chế Migration rất chặt chẽ (hàng trăm file migration trong thư mục `backend/src/db/migrations`).
*   **Caching & Queue:** **Redis** kết hợp với **BullMQ** hoặc **Pg-boss** để xử lý các tác vụ nền (như quét secret, gửi email, rotate keys).
*   **Validation:** **Zod** - Sử dụng để định nghĩa schema và validate dữ liệu ở cả tầng API và Database.

### Frontend (React & TypeScript)
*   **Build Tool:** **Vite** - Thay thế cho Webpack để có tốc độ phát triển nhanh hơn.
*   **Styling:** **Tailwind CSS**.
*   **State Management:** **React Query (TanStack Query)** để quản lý trạng thái server và caching.
*   **UI Components:** Sử dụng các thành phần tùy chỉnh (Custom Components) kết hợp với các thư viện như Radix UI.

### Infrastructures & DevOps
*   **Docker & Docker Compose:** Cung cấp nhiều cấu hình cho các môi trường khác nhau (dev, prod, bdd-test, read-replica).
*   **Kubernetes:** Hỗ trợ **Helm Charts** và **Kubernetes Operator** để tự động hóa việc đẩy secret vào K8s Workloads.
*   **Security:** Hỗ trợ chế độ **FIPS** (Federal Information Processing Standards) thông qua Dockerfile riêng biệt.

---

## 2. Kỹ thuật và Tư duy Kiến trúc

### Kiến trúc Đa tầng (Layered Architecture)
Backend được chia thành các lớp rõ rệt:
*   **DAL (Data Access Layer):** Lớp tương tác trực tiếp với Database thông qua Knex.
*   **Services:** Chứa logic nghiệp vụ xử lý secrets, mã hóa, và tích hợp bên thứ ba.
*   **Routes/Controllers:** Định nghĩa API endpoints bằng Fastify.

### Bảo mật "Zero Trust" & Mã hóa
*   **E2EE (End-to-End Encryption):** Bí mật thường được mã hóa ở phía client (hoặc tầng ứng dụng) trước khi lưu vào DB.
*   **KMS & HSM Integration:** Tích hợp với AWS KMS, GCP KMS và hỗ trợ module phần cứng bảo mật (HSM) qua giao thức PKCS11.
*   **Machine Identities:** Tư duy quản lý định danh không chỉ cho con người mà còn cho máy móc (Kubernetes Auth, AWS Auth, GCP Auth).

### Khả năng mở rộng (Extensibility)
Hệ thống được thiết kế dưới dạng module hóa cao:
*   **Integrations:** Dễ dàng mở rộng để đồng bộ (sync) secret sang Vercel, GitHub, AWS, Azure, v.v.
*   **Framework agnostic:** Hỗ trợ SDK cho nhiều ngôn ngữ (Node, Python, Go, Java...).

---

## 3. Các kỹ thuật chính nổi bật

1.  **Secret Rotation (Tự động đổi mật khẩu):** Hệ thống có khả năng tự động thay đổi mật khẩu định kỳ cho database (PostgreSQL, MySQL, Oracle) và các dịch vụ đám mây (AWS IAM).
2.  **Dynamic Secrets (Bí mật động):** Tạo ra các thông tin xác thực tạm thời (ephemeral) có thời hạn, tự động thu hồi sau khi sử dụng (ví dụ: tạo user DB tạm thời cho một job cụ thể).
3.  **Hạ tầng PKI nội bộ:** Tích hợp sẵn Certificate Authority (CA) để cấp phát và quản lý vòng đời chứng chỉ SSL/TLS nội bộ.
4.  **Secret Scanning:** Sử dụng kỹ thuật quét (scanning) để phát hiện và ngăn chặn việc đẩy nhầm secret lên các kho lưu trữ Git (như GitHub, GitLab).
5.  **Audit Logs:** Hệ thống ghi lại mọi hành động thay đổi hoặc truy cập bí mật để phục vụ việc tuân thủ (compliance) và bảo mật.
6.  **FIPS Compliance:** Cung cấp Dockerfile đặc biệt xây dựng dựa trên OpenSSL hỗ trợ FIPS, dành cho các doanh nghiệp yêu cầu tiêu chuẩn bảo mật chính phủ.

---

## 4. Tóm tắt luồng hoạt động (Workflow)

Hệ thống hoạt động theo một chu trình khép kín từ lúc khởi tạo đến khi phân phối:

1.  **Khởi tạo (Provisioning):** Quản trị viên thiết lập dự án (Project) và các môi trường (Dev, Staging, Prod). Các khóa mã hóa gốc (Root Key) được tạo ra.
2.  **Lưu trữ (Storage):** User nhập secret qua Dashboard hoặc CLI. Secret được mã hóa bằng thuật toán AES-GCM 256-bit trước khi lưu vào PostgreSQL.
3.  **Xác thực (Authentication):** Con người đăng nhập qua OIDC/SAML. Máy móc (Server/Bot) xác thực qua Machine Identity (như Kubernetes Service Account hoặc AWS IAM Role).
4.  **Phân phối (Delivery):** 
    *   Ứng dụng sử dụng **Infisical Agent** để nhận secret mà không cần sửa code.
    *   **Kubernetes Operator** theo dõi sự thay đổi của secret và tự động cập nhật K8s Secrets/ConfigMaps, sau đó restart deployment nếu cần.
    *   **Secret Sync** tự động đẩy secret sang các nền tảng khác (như biến môi trường trên Vercel).
5.  **Giám sát & Quản lý vòng đời:** Hệ thống tự động thực hiện **Rotation** nếu đến hạn, quét các repo Git để đảm bảo secret không bị lộ, và lưu lại lịch sử truy cập trong **Audit Logs**.

---
*Phân tích này dựa trên snapshot hiện tại của kho lưu trữ Infisical.*