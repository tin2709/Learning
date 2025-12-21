
# Quy trình CI/CD Framework (Tài liệu chi tiết)

Tài liệu này mô tả khung quy trình Tích hợp liên tục và Triển khai liên tục (CI/CD) hiện đại, giúp giảm thiểu sự phức tạp và tối ưu hóa vòng đời phát triển phần mềm.

## 1. Giới thiệu tổng quan
Quy trình CI/CD trong ảnh được thiết kế như một vòng lặp kín, tập trung vào việc tự động hóa, bảo mật (DevSecOps) và phản hồi liên tục để đảm bảo phần mềm luôn ở trạng thái sẵn sàng phát triển và vận hành tốt nhất.

---

## 2. Các giai đoạn chi tiết (Stages)

### 🟢 Giai đoạn 1: PLAN (Lập kế hoạch)
Đây là giai đoạn khởi đầu, nơi đội ngũ xác định những gì cần làm.
*   **Hoạt động chính:** Áp dụng phương pháp Agile (Scrum/Kanban), lập kế hoạch Sprint, phân chia tác vụ (Task breakdown) và ước lượng quy mô (Sizing).
*   **Yêu cầu:** Xác định các Yêu cầu phi chức năng (NFRs) ngay từ đầu để đảm bảo tính ổn định và bảo mật.
*   **Công cụ tiêu biểu:**
    *   Quản lý dự án: Jira Software, Trello, Asana, ServiceNow.

### 🔵 Giai đoạn 2: CODE (Lập trình)
Giai đoạn thực hiện viết mã nguồn dựa trên kế hoạch.
*   **Hoạt động chính:** Sử dụng IDE, quản lý mã nguồn qua Git, thực hiện Code Review và Pull Request.
*   **Nguyên tắc:** 
    *   **KISS (Keep It Simple, Stupid):** Viết code đơn giản, dễ bảo trì.
    *   **DevSecOps:** Đưa bảo mật vào ngay từ khi bắt đầu viết code.
*   **Công cụ tiêu biểu:**
    *   IDE: VS Code, Eclipse, IntelliJ.
    *   Quản lý nguồn: GitHub, Bitbucket, GitLab.
    *   Container (Môi trường dev): Docker, Vagrant.

### 🔴 Giai đoạn 3: BUILD (Xây dựng)
Khi code được đẩy lên (Push), hệ thống sẽ tự động build.
*   **Hoạt động chính:** Biên dịch mã nguồn, kiểm tra bảo mật sớm (Security checks), kiểm tra NFR và đóng gói thành Image (Docker build).
*   **Lưu ý:** Các tác vụ nhỏ giúp build nhanh hơn và giảm xung đột mã nguồn.
*   **Công cụ tiêu biểu:**
    *   Build Pipeline: Jenkins, Travis CI, CircleCI, GitLab CI, Bamboo.

### 🟡 Giai đoạn 4: TEST (Kiểm thử)
Kiểm tra chất lượng sản phẩm trước khi phát hành.
*   **Hoạt động chính:** Unit testing, Integration testing, Security testing, quét lỗ hổng (CVE), và kiểm tra tính tuân thủ bản quyền.
*   **Nguyên tắc:** **TDD (Test-Driven Development)** - Viết test trước khi viết code.
*   **Công cụ tiêu biểu:**
    *   Testing: SonarQube, ESLint, JS Hint.
    *   Bảo mật: Aqua, Sysdig, Twistlock.

> **Lưu ý:** Nếu giai đoạn **BUILD** hoặc **TEST** thất bại, quy trình sẽ quay trở lại giai đoạn **CODE** để sửa lỗi.

### 🟠 Giai đoạn 5: RELEASE (Phát hành)
Đưa sản phẩm đã qua kiểm tra vào kho lưu trữ sẵn sàng triển khai.
*   **Hoạt động chính:** Gắn tag phiên bản, tạo Release notes, cập nhật tài liệu tự động và chuyển giao vận hành.
*   **Công cụ tiêu biểu:**
    *   Kho lưu trữ: Artifactory, Nexus.
    *   Container Registry: Docker Hub, Quay.io, Amazon ECR, Google Container Registry.

### 🟣 Giai đoạn 6: DEPLOY (Triển khai)
Đưa sản phẩm đến môi trường chạy thực tế hoặc môi trường trung gian.
*   **Hoạt động chính:** Di chuyển traffic, triển khai Canary hoặc A/B testing để giảm thiểu rủi ro. Xác thực tính năng trên môi trường thực tế.
*   **Công cụ tiêu biểu:**
    *   Tự động hóa triển khai: Helm, Consul, Terraform, AWS CloudFormation.

### 🔘 Giai đoạn 7: OPERATE (Vận hành)
Quản lý sản phẩm đang chạy.
*   **Hoạt động chính:** Quy trình vận hành, quản lý sự cố (On-call), cải thiện tài liệu và thu thập phản hồi từ người dùng.
*   **Công cụ tiêu biểu:**
    *   Quản lý cấu hình: Ansible, Puppet, Chef, Saltstack.
    *   Quản lý khóa/bí mật: HashiCorp Vault.
    *   Service Mesh: Istio, Envoy, NGINX.

### 🟡 Giai đoạn 8: MONITOR & OPTIMIZE (Giám sát & Tối ưu hóa)
Đây là giai đoạn cuối nhưng diễn ra liên tục để cải thiện hệ thống.
*   **Hoạt động chính:**
    *   **Monitor:** Giám sát hiệu năng, tính khả dụng (Observability) và phản hồi NFR.
    *   **Optimize:** Tự động tối ưu hóa tài nguyên dựa trên phân tích dữ liệu (Machine Learning), điều chỉnh kích thước cơ sở hạ tầng.
*   **Công cụ tiêu biểu:**
    *   Giám sát: Dynatrace, Prometheus, Datadog, New Relic, Splunk.
    *   Điều phối (Orchestration): Kubernetes (K8s), Docker Swarm, Nomad.
    *   Tối ưu hóa: **Densify**.

---

## 3. Tổng kết Công cụ theo danh mục (Tool Stack)

| Danh mục | Công cụ phổ biến trong ảnh |
| :--- | :--- |
| **Agile/Scrum** | Jira, Trello, Asana, ServiceNow |
| **Source Control** | GitHub, GitLab, Bitbucket |
| **CI/CD Pipelines** | Jenkins, CircleCI, Travis CI, Bamboo |
| **Infrastructure as Code** | Terraform, CloudFormation, Pulumi |
| **Security/Compliance** | SonarQube, Aqua, Sysdig, Falco |
| **Container/Registry** | Docker, Harbor, Quay, ECR, GCR |
| **Monitoring/Logging** | Prometheus, Grafana, Splunk, Datadog |
| **Orchestration** | Kubernetes, OpenShift, Rancher |

---
*Tài liệu được tổng hợp dựa trên Framework của Densify 2019.*