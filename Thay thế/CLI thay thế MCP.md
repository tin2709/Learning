Dưới đây là nội dung bài viết được chuyển đổi sang định dạng file `README.md` để bạn có thể lưu trữ trên GitHub hoặc các hệ thống quản lý tài liệu kỹ thuật.

---

# MCP vs CLI: Tại sao Lập trình viên đang quay lại với Terminal cho AI Agent?

![AI Agent Trend](https://img.shields.io/badge/Trend-AI--Agent-blue)
![Category](https://img.shields.io/badge/Category-DevTools-green)
![Status](https://img.shields.io/badge/Status-Trending-red)

> **"MCP is dead. Long live the CLI."** — *Eric Holmes*

Từng được kỳ vọng là "HTTP của kỷ nguyên AI", **Model Context Protocol (MCP)** đang đối mặt với làn sóng quay xe từ cộng đồng lập trình viên. Thay vì các giao thức phức tạp, xu hướng hiện tại đang dịch chuyển mạnh mẽ về **CLI-first** (Ưu tiên giao diện dòng lệnh).

## 📖 Mục lục
- [Giới thiệu](#-giới-thiệu)
- [Vấn đề của MCP](#-vấn-đề-của-mcp-3-cái-bẫy)
- [Sức mạnh của CLI đối với AI](#-tại-sao-cli-là-thứ-tự-nhiên-nhất-với-ai-agent)
- [So sánh chi tiết](#-so-sánh-mcp-vs-cli)
- [Workflow tiêu biểu](#-workflow-kinh-điển-của-cli-agent)
- [Khi nào nên dùng MCP?](#-khi-nào-mcp-vẫn-tỏa-sáng)
- [Kết luận](#-tóm-gọn-lại)

---

## 🚀 Giới thiệu
MCP được thiết kế để chuẩn hóa cách AI Agent sử dụng các công cụ bên ngoài thông qua Schema và Context. Tuy nhiên, khi đưa vào thực tế phát triển phần mềm, nó bộc lộ những rào cản về hiệu suất và độ phức tạp, dẫn đến sự trỗi dậy của các AI Agent thuần CLI (như Claude Code).

---

## ⚠️ Vấn đề của MCP (3 cái bẫy)

### 1. Hao hụt Token (Overhead)
Để AI hiểu được một Tool trong MCP, bạn phải gửi kèm: Tên công cụ, mô tả chi tiết, và Schema JSON của tham số.
- **Hệ quả:** Nếu có hàng chục dịch vụ, hàng nghìn Token sẽ bị tiêu tốn chỉ để AI "đọc hướng dẫn sử dụng" trước khi bắt đầu làm việc.

### 2. "Phát minh lại bánh xe"
Đa số MCP Server hiện nay chỉ là các Wrapper (lớp bọc) cho các công cụ đã có sẵn (Docker, Git, AWS CLI, Kubectl). 
- **Hệ quả:** Thêm một layer MCP chỉ làm tăng nợ kỹ thuật và công sức bảo trì.

### 3. Phá vỡ triết lý Unix
Triết lý Unix là sự kết hợp các công cụ nhỏ qua đường ống (Pipeline). MCP biến các thao tác linh hoạt này thành các cuộc gọi hàm (Function call) đơn lẻ, rời rạc và cứng nhắc.

---

## 🛠️ Tại sao CLI là thứ "tự nhiên" nhất với AI Agent?

- **Dữ liệu huấn luyện khổng lồ:** LLM đã được học qua hàng triệu dòng lệnh Bash, tài liệu DevOps và thảo luận trên Stack Overflow. Chúng hiểu CLI một cách bản năng.
- **Tính tái lập (Reproducibility):** Khi Agent lỗi, bạn chỉ cần copy lệnh CLI đó chạy lại trên Terminal để debug. Với MCP, bạn phải lục lọi JSON Payload và Logs hệ thống.
- **Khả năng lắp ghép:** Agent có thể tự do dùng `grep`, `jq`, `awk` để xử lý dữ liệu giữa các lệnh mà không cần định nghĩa trước Schema.

---

## 📊 So sánh MCP vs CLI

| Đặc điểm | MCP (Protocol-based) | CLI (Command-based) |
| :--- | :--- | :--- |
| **Cấu trúc** | Rõ ràng, Type-safe | Linh hoạt, Text-based |
| **Chi phí Token** | Cao (do Schema/Description) | Thấp (chỉ cần tên lệnh) |
| **Độ phức tạp** | Cao (cần Server/Client) | Thấp (chạy trực tiếp) |
| **Khả năng Debug** | Khó (phụ thuộc log hệ thống) | Dễ (tái lập 1:1 trên Terminal) |
| **Hệ sinh thái** | Đang xây dựng | Đã hoàn thiện hàng thập kỷ |

---

## 💻 Workflow kinh điển của CLI Agent

### Tác vụ Refactor Code:
```bash
# Agent tự thực thi chuỗi lệnh
rg "oldDeprecatedFunction" .
git diff
npm test
git commit -m "Refactor deprecated functions"
```

### Điều tra & Fix bug:
```bash
tail -f logs/server.log | grep error
curl -v api/auth/check
docker-compose up -d
```

---

## 💡 Khi nào MCP vẫn tỏa sáng?
MCP không hoàn toàn vô dụng. Nó cực kỳ hiệu quả trong các trường hợp:
- **Hệ thống SaaS doanh nghiệp:** Tích hợp các API đóng, bảo mật cao.
- **Ứng dụng cho người dùng cuối:** Những người không biết sử dụng Terminal.
- **Rào chắn bảo mật:** Cần kiểm soát chặt chẽ quyền truy cập thông qua các Gateway cứng.

---

## 📝 Tóm gọn lại
AI không tạo ra một trật tự phát triển mới, nó chỉ đang "khai quật" lại giá trị bất diệt của **Terminal**. Chiếc Terminal mộc mạc, linh hoạt và mạnh mẽ chính là "thanh gươm" bén nhất cho các AI Agent hiện nay.

---

## 🔗 Tham khảo
- [MCP is dead. Long live the CLI (Blog)](https://x.com/levelsio/status/2031943074151104634)
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code)
- [Reddit Discussion: MCP vs CLI Tools](https://www.reddit.com/r/ClaudeAI/comments/1o99i6y/mcp_vs_cli_tools/)

---
*Nội dung dựa trên bài chia sẻ của **Sky blue @takuya** trên Viblo.*