Dựa trên các tệp tin được cung cấp, dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc, kỹ thuật lập trình và luồng hoạt động của dự án **RustNet**.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

RustNet không chỉ là một wrapper cho `libpcap`, mà là một hệ thống phân tích mạng phức tạp kết hợp nhiều công nghệ cấp thấp:

*   **Capture Engine (Lớp bắt gói tin):** Sử dụng `libpcap` làm cơ sở, nhưng mở rộng theo nền tảng:
    *   **Linux:** eBPF (`libbpf-rs`) để theo dõi socket trực tiếp từ kernel (hiệu năng cao nhất).
    *   **macOS:** Tận dụng `PKTAP` (Packet Tap) để trích xuất metadata tiến trình trực tiếp từ header gói tin (độc quyền Apple).
    *   **Windows:** Sử dụng Npcap và IP Helper API (`GetExtendedTcpTable`).
*   **Deep Packet Inspection (DPI):** Tự triển khai các parser cho giao thức (HTTP, TLS/SNI, SSH, QUIC, MQTT, DNS...) thay vì dựa vào thư viện bên ngoài lớn, giúp duy trì binary nhỏ gọn và tốc độ cực nhanh.
*   **Bảo mật (Defense-in-Depth):**
    *   **Landlock (Linux):** Sandboxing mức kernel để giới hạn quyền truy cập file/mạng ngay sau khi khởi tạo.
    *   **Seatbelt (macOS):** Sử dụng `sandbox_init` để cô lập tiến trình.
    *   **Restricted Token (Windows):** Hạ cấp đặc quyền token tiến trình.
*   **Giao diện TUI:** `ratatui` kết hợp với `crossterm`, hỗ trợ Braille đồ họa (sparklines) và tương tác chuột.

### 2. Tư duy Kiến trúc (Architectural Logic)

Kiến trúc của RustNet được thiết kế theo mô hình **Pipeline đa luồng không chặn (Lock-free/Concurrent Pipeline)**:

*   **Thiết kế Snapshot-Isolation:** Luồng UI không bao giờ truy cập trực tiếp vào dữ liệu đang thay đổi. Thay vào đó, nó sử dụng một `SnapshotProvider` để tạo bản sao (`clone`) dữ liệu từ `DashMap` định kỳ (mặc định 1s). Điều này giúp UI mượt mà ngay cả khi xử lý hàng chục nghìn gói tin mỗi giây.
*   **Quản lý trạng thái bằng DashMap:** Thay vì dùng một `Mutex<HashMap>` (gây hiện tượng thắt nút cổ chai), dự án dùng `DashMap` (Sharded Hashmap). Các luồng xử lý gói tin có thể ghi vào các "shard" khác nhau đồng thời mà không chờ đợi nhau.
*   **Module hóa theo Platform:** Cấu trúc thư mục `src/network/platform/` được chia rõ ràng theo hệ điều hành. Sử dụng `Conditional Compilation` (`#[cfg(target_os = ...)]`) triệt để để tối ưu hóa binary cho từng nền tảng.
*   **Intelligent Timeout System:** Kiến trúc quản lý vòng đời kết nối dựa trên ngữ cảnh giao thức (ví dụ: SSH giữ 30 phút, DNS chỉ 30 giây).

### 3. Kỹ thuật Lập trình Đặc sắc (Programming Techniques)

*   **Zero-cost Abstractions trong DPI:** Sử dụng các kỹ thuật trích xuất dữ liệu trực tiếp từ buffer byte (`&[u8]`) mà không cần cấp phát lại bộ nhớ (allocation) giúp giảm áp lực lên bộ dọn rác (GC-free performance).
*   **CoW (Copy on Write) trong Rate Tracking:** Trong module `RateTracker`, dữ liệu mẫu (samples) được lưu trong `Arc<VecDeque>`. Khi cần cập nhật, nó chỉ copy nếu có luồng khác đang giữ snapshot (`Arc::make_mut`), tối ưu hóa việc đọc dữ liệu để vẽ biểu đồ.
*   **Trình tối ưu hóa Build (build.rs):** File `build.rs` cực kỳ phức tạp, tự động tải Npcap SDK trên Windows, biên dịch chương trình eBPF C trên Linux và tạo shell completions. Đây là kỹ thuật "Automation-at-build" điển hình của Rust chuyên nghiệp.
*   **An toàn bộ nhớ (Safety First):** Mặc dù làm việc với con trỏ và syscall hệ thống (đặc biệt trong phần eBPF và PKTAP), dự án bọc các phần `unsafe` trong các interface an toàn, có tài liệu giải thích rõ ràng (như thấy trong `CHANGELOG.md` bản 1.1.0).

### 4. Luồng Hoạt động Hệ thống (System Data Flow)

1.  **Giai đoạn Khởi động:**
    *   Kiểm tra đặc quyền (Capabilities trên Linux / Root trên Mac).
    *   Mở handle capture (`libpcap` / `socket`).
    *   **Sandbox Activation:** Ngay lập tức kích hoạt Landlock/Seatbelt để tự khóa mình vào "nhà tù" an toàn.
2.  **Luồng Capture (Producer):**
    *   Gói tin thô được đọc -> Đóng gói vào `ParsedPacket`.
    *   Đẩy vào `crossbeam::channel` (MPSC).
3.  **Luồng Xử lý (Worker):**
    *   Pop gói tin từ channel.
    *   DPI Engine bóc tách lớp ứng dụng (SNI, HTTP Host, v.v.).
    *   Tra cứu GeoIP và Service mapping (từ các file nhúng `include_bytes!`).
    *   Cập nhật `DashMap<ConnectionKey, Connection>`.
4.  **Luồng Làm giàu (Enrichment):**
    *   Định kỳ (mỗi vài giây), hệ thống gọi API hệ thống (procfs, lsof, GetExtendedTable) để map `Socket Inode` -> `PID` -> `Process Name`.
5.  **Luồng Cleanup & UI (Consumer):**
    *   Cleanup thread duyệt map, xóa các kết nối hết hạn.
    *   UI thread lấy snapshot, áp dụng bộ lọc (Regex/Fzf-style) và render lên terminal qua `ratatui`.

### 5. Điểm nhấn về Hiệu năng (Performance Benchmarking)

Nhìn vào thư mục `benches/`, chúng ta thấy tác giả chú trọng vào:
*   **Packet Parsing:** Đo lường tốc độ phân tích từng byte.
*   **Connection Merge:** Tối ưu hóa việc gộp dữ liệu gói tin vào kết nối hiện có (đạt mức nano-giây).
*   **Snapshot Creation:** Đảm bảo việc clone hàng nghìn kết nối để hiển thị không gây lag hệ thống.

**Tổng kết:** RustNet là một ví dụ mẫu mực về việc sử dụng Rust để xây dựng công cụ hệ thống hiện đại: **Nhanh như C, an toàn như Java và tiện dụng như Python.** Luồng xử lý dữ liệu tách biệt giữa capture, enrichment và hiển thị là điểm mấu chốt giúp nó vượt qua các công cụ truyền thống như `netstat` hay `iftop`.