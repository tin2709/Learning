Dưới đây là bản phân tích chi tiết về dự án **Scrutiny** và tóm tắt nội dung dưới dạng tệp README bằng tiếng Việt.

---

### 1. Phân tích Công nghệ Cốt lõi
Dự án được xây dựng dựa trên sự kết hợp giữa hiệu suất của hệ thống và trải nghiệm người dùng hiện đại:
*   **Ngôn ngữ Backend:** **Go (Golang)**. Lựa chọn này giúp tối ưu hóa việc giao tiếp với hệ điều hành (thông qua CLI `smartctl`), hỗ trợ đa nền tảng tốt và tạo ra các file thực thi (binary) tĩnh nhẹ nhàng.
*   **Framework Frontend:** **Angular (v13+)**. Sử dụng kiến trúc component mạnh mẽ, RxJS để xử lý luồng dữ liệu bất đồng bộ và Tailwind CSS để tùy biến giao diện nhanh chóng.
*   **Cơ sở dữ liệu (Hybrid):**
    *   **SQLite (GORM):** Dùng để lưu trữ các thông tin cấu hình, danh mục thiết bị và các dữ liệu quan hệ ổn định.
    *   **InfluxDB (v2.2):** Dùng lưu trữ dữ liệu chuỗi thời gian (time-series) như nhiệt độ và các chỉ số S.M.A.R.T theo thời gian thực để vẽ biểu đồ xu hướng.
*   **Công cụ hệ thống:** Dựa hoàn toàn trên **smartmontools (`smartctl`)**, tiêu chuẩn công nghiệp về kiểm tra sức khỏe ổ cứng.
*   **Thông báo:** Sử dụng thư viện **Shoutrrr**, cho phép tích hợp hàng chục dịch vụ (Discord, Telegram, Slack, Email...) chỉ với một dòng cấu hình URL.

### 2. Tư duy Kiến trúc (Architecture Thinking)
Kiến trúc của Scrutiny được thiết kế theo mô hình **Hub/Spoke (Trục và Nan hoa)**:
*   **Kiến trúc Monorepo:** Quản lý cả Backend, Frontend và Collector trong một kho lưu trữ duy nhất giúp đồng bộ hóa phiên bản dễ dàng.
*   **Mô hình Phân tán:**
    *   **Collector (Spoke):** Là một tác vụ nhẹ (Agent) chạy trên mọi server cần giám sát. Nó thu thập dữ liệu và "đẩy" (push) về trung tâm qua API. Điều này cho phép giám sát hàng trăm máy chủ từ một giao diện duy nhất.
    *   **Web/API (Hub):** Đóng vai trò là bộ não, tiếp nhận dữ liệu, lưu trữ vào DB và cung cấp giao diện Web cho người dùng.
*   **Containerization:** Sử dụng **S6-overlay** trong Docker (Omnibus image) để quản lý nhiều tiến trình (Web server, InfluxDB, Cron) chạy song song một cách ổn định trong một container duy nhất.

### 3. Các Kỹ thuật Chính (Key Techniques)
*   **Downsampling (Giảm mật độ dữ liệu):** Kỹ thuật quan trọng để tránh việc cơ sở dữ liệu phình to quá mức. Scrutiny tự động gộp các điểm dữ liệu cũ (ví dụ: từ dữ liệu từng giờ thành dữ liệu trung bình ngày/tuần) để duy trì lịch sử lâu dài mà không tốn dung lượng.
*   **Failure Prediction (Dự báo lỗi):** Không chỉ dựa vào ngưỡng (threshold) của nhà sản xuất, Scrutiny tích hợp dữ liệu tỷ lệ lỗi thực tế từ **Backblaze** để cảnh báo sớm các dấu hiệu hỏng hóc mà SMART thông thường có thể bỏ qua.
*   **Multi-arch Build:** Quy trình CI/CD qua GitHub Actions được thiết lập để build cho hầu hết các kiến trúc CPU (amd64, arm64, armv7...), hỗ trợ từ PC cá nhân đến các dòng NAS chạy chip ARM.
*   **Udev Integration:** Ánh xạ metadata thiết bị từ host vào container qua `/run/udev`, giúp định danh ổ cứng chính xác kể cả khi tên thiết bị (`/dev/sda`, `/dev/sdb`) thay đổi sau khi khởi động lại.

### 4. Tóm tắt luồng hoạt động
1.  **Quét (Scan):** Collector chạy lệnh `smartctl --scan` để phát hiện tất cả ổ cứng có trong hệ thống.
2.  **Thu thập (Collect):** Theo lịch trình (Cron), Collector lấy thông tin chi tiết (JSON) của từng ổ đĩa.
3.  **Gửi dữ liệu:** Collector gửi bản tin JSON này tới API Backend.
4.  **Xử lý & Lưu trữ:** Backend phân tích dữ liệu, ghi thông tin ổ đĩa vào SQLite và ghi các chỉ số SMART vào InfluxDB.
5.  **Cảnh báo:** Nếu phát hiện chỉ số vượt ngưỡng nguy hiểm, hệ thống kích hoạt Shoutrrr để gửi thông báo ngay lập tức.
6.  **Hiển thị:** Người dùng truy cập Web UI để xem bảng điều hành tổng quát về sức khỏe ổ cứng và biểu đồ lịch sử.

---

# [README.vi.md] - Scrutiny: Hệ thống Giám sát Sức khỏe Ổ cứng

## 📋 Giới thiệu
**Scrutiny** là một giải pháp Dashboard giám sát sức khỏe ổ cứng (S.M.A.R.T) hiện đại, tập trung vào việc theo dõi xu hướng lịch sử và dự báo tỷ lệ hỏng hóc dựa trên dữ liệu thực tế.

Dự án khắc phục nhược điểm của các công cụ dòng lệnh truyền thống bằng cách cung cấp giao diện Web trực quan và hệ thống lưu trữ dữ liệu chuỗi thời gian mạnh mẽ.

## 🚀 Tính năng nổi bật
*   **Giao diện Dashboard Web:** Theo dõi tập trung trạng thái của tất cả ổ cứng.
*   **Phân tích thông minh:** Kết hợp ngưỡng lỗi của nhà sản xuất với dữ liệu tỷ lệ hỏng hóc thực tế từ Backblaze.
*   **Theo dõi xu hướng:** Lưu trữ lịch sử nhiệt độ và các thuộc tính SMART để phát hiện sự xuống cấp dần theo thời gian.
*   **Kiến trúc Hub/Spoke:** Giám sát nhiều máy chủ từ xa từ một trung tâm duy nhất.
*   **Hệ thống thông báo đa dạng:** Hỗ trợ Discord, Telegram, Slack, Email và nhiều dịch vụ khác qua Webhooks.
*   **Hỗ trợ RAID:** Tương thích với hầu hết các bộ điều khiển RAID hỗ trợ `smartctl`.

## 🛠 Công nghệ sử dụng
*   **Backend:** Golang (Gin, GORM, CLI smartctl)
*   **Frontend:** Angular 13, Tailwind CSS, ApexCharts
*   **Database:** InfluxDB v2 (dữ liệu lịch sử), SQLite (cấu hình & metadata)
*   **DevOps:** Docker (Omnibus & Hub/Spoke), GitHub Actions (Multi-arch build)

## 🏗 Kiến trúc hệ thống
Hệ thống gồm 2 thành phần chính:
1.  **Collector (Spoke):** Chạy trên từng máy chủ, thu thập dữ liệu từ ổ cứng và đẩy về Web API.
2.  **Webapp & API (Hub):** Tiếp nhận dữ liệu, phân tích, lưu trữ vào cơ sở dữ liệu và hiển thị giao diện người dùng.

## 📦 Hướng dẫn cài đặt nhanh (Docker)
Cách nhanh nhất để chạy Scrutiny (Omnibus - All in one):

```bash
docker run -p 8080:8080 -p 8086:8086 \
  -v `pwd`/scrutiny:/opt/scrutiny/config \
  -v `pwd`/influxdb:/opt/scrutiny/influxdb \
  -v /run/udev:/run/udev:ro \
  --cap-add SYS_RAWIO \
  --cap-add SYS_ADMIN \
  --device=/dev/sda \
  --device=/dev/sdb \
  --name scrutiny \
  ghcr.io/analogj/scrutiny:master-omnibus
```
*(Lưu ý: Thay thế `--device` bằng các đường dẫn ổ cứng thực tế trên máy của bạn).*

## 📈 Cơ chế xử lý dữ liệu
Scrutiny sử dụng kỹ thuật **Downsampling** để tối ưu dung lượng lưu trữ:
*   Dữ liệu chi tiết được giữ trong 15 ngày.
*   Dữ liệu gộp theo tuần giữ trong 9 tuần.
*   Dữ liệu gộp theo tháng giữ trong 25 tháng.
*   Dữ liệu gộp theo năm được giữ vĩnh viễn.

## 🤝 Đóng góp
Dự án được phát triển chính bởi **Jason Kulatunga (@AnalogJ)** và cộng đồng. Mọi đóng góp về code hoặc báo lỗi vui lòng thực hiện qua GitHub Issues/Pull Requests.

---
*Tài liệu này được tóm tắt dựa trên mã nguồn và tài liệu chính thức của dự án Scrutiny.*