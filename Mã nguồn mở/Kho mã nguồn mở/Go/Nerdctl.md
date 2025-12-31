Dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án **nerdctl** dựa trên mã nguồn bạn đã cung cấp.

---

# Báo cáo Phân tích Dự án nerdctl

## 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Go (Golang)** - chiếm 96.6% mã nguồn, tận dụng khả năng xử lý đồng thời và tính gọn nhẹ của Go cho các công cụ hệ thống.
*   **Container Engine:** **containerd** - nerdctl đóng vai trò là client CLI chính thức (non-core) để tương tác trực tiếp với containerd thông qua gRPC API.
*   **CLI Framework:** **Cobra** & **Pflag** - Sử dụng để xây dựng giao diện dòng lệnh, quản lý flag và sub-commands (tương tự như Docker và Kubernetes).
*   **Container Runtime:** **runc** (mặc định) hoặc các runtime tương thích OCI khác.
*   **Networking:** **CNI (Container Network Interface)** - Quản lý mạng cho container thay vì sử dụng cơ chế bridge riêng của Docker.
*   **Build Engine:** **BuildKit** - Tích hợp để thực hiện lệnh `nerdctl build`, hỗ trợ tính năng build hiện đại và hiệu năng cao.
*   **Cơ chế lưu trữ (Storage):** Sử dụng các **Snapshotter** của containerd như overlayfs, btrfs, và đặc biệt hỗ trợ các snapshotter tiên tiến (Stargz, Nydus, SOCI) để kéo ảnh (pull image) tốc độ cao.

## 2. Tư duy kiến trúc (Architectural Thinking)

*   **Tính tương thích tối đa (Docker-compatibility):** Kiến trúc được thiết kế để người dùng Docker có thể chuyển sang mà không cần học lại lệnh (`nerdctl run`, `nerdctl ps`, `nerdctl compose`...).
*   **Kiến trúc Client-only (Daemonless-ish):** Khác với Docker CLI (giao tiếp với Docker Daemon), nerdctl giao tiếp trực tiếp với containerd daemon. Nó không cần một "nerdctl daemon" riêng biệt.
*   **Phân tách Namespace:** Sử dụng tính năng namespace của containerd để cô lập dữ liệu. Mặc định dùng namespace `default`, nhưng có thể tương tác với `k8s.io` để debug các container của Kubernetes.
*   **Thiết kế Rootless-first:** Hỗ trợ chạy container mà không cần quyền root thông qua `RootlessKit` và `slirp4netns`, hướng tới bảo mật tối đa.
*   **Tính module hóa (Extensibility):** Các tính năng nâng cao như mã hóa hình ảnh (`ocicrypt`), ký tên (`cosign`), hay phân phối P2P (`IPFS`) được thiết kế dưới dạng các module tùy chọn (optional).

## 3. Các kỹ thuật chính (Key Techniques)

*   **Lazy Pulling (Kéo ảnh lười):** Sử dụng Stargz/Nydus để chạy container ngay lập tức mà không cần chờ tải xong toàn bộ image (chỉ tải những file cần thiết khi container yêu cầu).
*   **Compose Integration:** Tích hợp trực tiếp thư viện `compose-go` để đọc file `docker-compose.yaml` và thực hiện các thao tác quản lý nhiều container mà không cần cài thêm docker-compose.
*   **Mã hóa Image:** Kỹ thuật dùng `imgcrypt` (dựa trên OCIcrypt) để mã hóa các layer của image, đảm bảo nội dung image chỉ được đọc trên các host có key giải mã.
*   **Bypass4netns:** Kỹ thuật tối ưu hóa mạng cho chế độ rootless, giảm thiểu chi phí xử lý (overhead) của slirp4netns, giúp tăng tốc độ truyền tải dữ liệu.
*   **Dual-format Export:** Hỗ trợ xuất image ra cả định dạng Docker v1.2 và OCI v1.0, giúp linh hoạt trong việc di chuyển image giữa các hệ thống khác nhau.

---

# Luồng hoạt động của Dự án (README Summary)

## Giới thiệu
**nerdctl** là một công cụ dòng lệnh (CLI) tương thích với Docker dành cho **containerd**. Nó cung cấp trải nghiệm người dùng giống hệt Docker nhưng hỗ trợ các tính năng tiên tiến nhất của hệ sinh thái containerd.

## Luồng hoạt động chính (Workflow)

1.  **Tiếp nhận yêu cầu (CLI Layer):**
    *   Người dùng nhập lệnh (ví dụ: `nerdctl run -d --name nginx nginx:alpine`).
    *   Thư viện **Cobra** phân tích lệnh, kiểm tra các flag toàn cục (như `--namespace`, `--address`) và các tham số riêng của lệnh.

2.  **Cấu hình và Khởi tạo (Initialization):**
    *   Tải cấu hình từ file `nerdctl.toml`.
    *   Nếu đang chạy ở chế độ **Rootless**, nerdctl tự động thiết lập các namespace về user thông qua `RootlessKit`.
    *   Khởi tạo gRPC client để kết nối tới socket của containerd (thường là `/run/containerd/containerd.sock`).

3.  **Xử lý Tài nguyên (Resource Management):**
    *   **Image:** Nếu image chưa có cục bộ, nerdctl gọi containerd để kéo image. Nếu có snapshotter đặc biệt (như Stargz), nó sẽ thực hiện kéo ảnh tối ưu.
    *   **Network:** Sử dụng các plugin CNI để tạo interface mạng và cấp phát IP cho container.
    *   **Volume:** Tạo các thư mục quản lý dữ liệu trong `/var/lib/nerdctl`.

4.  **Thực thi Container (Execution):**
    *   nerdctl gửi yêu cầu tạo Spec OCI tới containerd.
    *   containerd yêu cầu **containerd-shim** khởi tạo container thông qua **runc**.
    *   nerdctl theo dõi trạng thái, thu thập log và hiển thị cho người dùng.

5.  **Tính năng Compose (Multi-container):**
    *   Khi dùng `nerdctl compose up`, công cụ sẽ đọc file YAML, phân tích cây phụ thuộc giữa các dịch vụ.
    *   Nó tuần tự thực hiện các bước tạo mạng, pull image và khởi chạy từng container theo đúng thứ tự logic.

## Cách cài đặt nhanh
*   **Linux:** Tải bản binary `nerdctl-full` (bao gồm đầy đủ containerd, runc, CNI, BuildKit).
*   **macOS:** Sử dụng thông qua dự án **Lima** (`brew install lima`).
*   **Windows:** Hỗ trợ thông qua WSL2.

## Lợi ích so với Docker CLI truyền thống
*   Hỗ trợ trực tiếp các tính năng mới nhất của containerd (Lazy pulling, IPFS, mã hóa image).
*   Nhẹ hơn, không phụ thuộc vào Docker Daemon.
*   Khả năng can thiệp sâu vào các namespace của Kubernetes để debug.