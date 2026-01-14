Dựa trên các tệp tin bạn đã cung cấp, đây là phân tích chi tiết về dự án **SoulSync** — một nền tảng tự động hóa và khám phá âm nhạc thông minh dành cho các máy chủ tự lưu trữ (self-hosted).

---

### 1. Công nghệ cốt lõi (Tech Stack)
*   **Ngôn ngữ lập trình:** Python 3.8+ (chiếm ~65% mã nguồn).
*   **Web Framework:** Flask (dùng cho WebUI và API endpoint).
*   **Cơ sở dữ liệu:** SQLite (lưu trữ metadata, cấu hình, wishlist và trạng thái thư viện).
*   **Containerization:** Docker & Docker Compose (hỗ trợ đa kiến trúc AMD64/ARM64), tối ưu cho Unraid.
*   **Giao diện người dùng:** HTML/JS/CSS (WebUI) và PyQt6 (Desktop GUI - hiện đang ở chế độ bảo trì).
*   **Tích hợp bên thứ ba:**
    *   **Streaming:** Spotify API, Tidal API.
    *   **Nguồn tải:** slskd (Soulseek), yt-dlp (YouTube), Beatport.
    *   **Media Servers:** Plex, Jellyfin, Navidrome.
    *   **Metadata/Lyrics:** LRClib, Music-map, ListenBrainz.

### 2. Kỹ thuật và Tư duy Kiến trúc
*   **Kiến trúc Orchestration (Điều phối):** Dự án sử dụng `DownloadOrchestrator` để quản lý đa nguồn tải. Nó cho phép người dùng chọn chế độ: Chỉ Soulseek, Chỉ YouTube, hoặc Hybrid (ưu tiên Soulseek, lỗi thì chuyển YouTube).
*   **Cơ chế Trì hoãn & Chống rung (Debouncing):** `MediaScanManager` được thiết kế rất thông minh. Thay vì quét thư viện Plex/Jellyfin ngay lập tức khi mỗi bài hát tải xong (gây nghẽn server), nó đợi một khoảng thời gian (debounce) để gom cụm các yêu cầu quét lại.
*   **Quản lý cấu hình linh hoạt:** `ConfigManager` hỗ trợ di chuyển từ tệp `config.json` sang lưu trữ trong database SQLite, giúp thay đổi cài đặt thời gian thực mà không cần khởi động lại container.
*   **Xử lý bất đồng bộ (Asyncio):** Sử dụng `asyncio` và `aiohttp` để tìm kiếm đồng thời trên hàng chục người dùng Soulseek, tối ưu hóa tốc độ phản hồi.

### 3. Các kỹ thuật chính nổi bật
*   **Matching Engine (Công cụ so khớp):**
    *   Sử dụng `SequenceMatcher` và `unidecode` để xử lý các tên nghệ sĩ có ký tự đặc biệt (ví dụ: `KoЯn`, `Björk`, `A$AP Rocky`).
    *   **Phân tích phiên bản (Version Detection):** Một kỹ thuật rất sâu giúp phân biệt bài hát gốc với bản Remix, Live, Acoustic hoặc Radio Edit dựa trên từ khóa trong tên tệp, tránh việc tải nhầm phiên bản người dùng không muốn.
*   **Tổ chức tệp dựa trên Template:** Cho phép người dùng tùy biến cấu trúc thư mục thông qua các biến như `$albumartist`, `$album`, `$track`. Hệ thống tự động dọn dẹp và di chuyển tệp từ thư mục tải về sang thư mục thư viện chính thức.
*   **Metadata Enhancement:** Tự động nhúng bìa album (Pillow) và tải tệp lời bài hát `.lrc` đồng bộ (lyrics_client.py) để hiển thị lời trên các thiết bị nghe nhạc.

### 4. Tóm tắt luồng hoạt động của dự án
1.  **Khám phá (Discovery):** SoulSync theo dõi nghệ sĩ trên Spotify/Tidal hoặc quét các bảng xếp hạng Beatport.
2.  **Kiểm tra (Matching):** Hệ thống so sánh với thư viện hiện tại (Plex/Jellyfin) để xác định bài nào còn thiếu.
3.  **Tìm kiếm (Search):** `matching_engine` tạo ra nhiều biến thể truy vấn (Query variations) để tìm kiếm trên Soulseek/YouTube.
4.  **Tải về (Download):** Gửi lệnh đến `slskd` hoặc `yt-dlp`. Nếu tải lỗi, bài hát được đưa vào **Wishlist** để tự động thử lại sau mỗi 30 phút.
5.  **Hậu xử lý (Post-processing):** Gắn tag metadata, tải lời bài hát, đổi tên tệp theo template và di chuyển vào thư viện nhạc.
6.  **Cập nhật (Sync):** Thông báo cho Media Server (Plex/Jellyfin) cập nhật thư viện mới.

### 5. Điểm mạnh của dự án
*   **Khả năng chịu lỗi cao:** Có hệ thống retry tự động và logic fallback giữa các nguồn tải.
*   **Cá nhân hóa:** Tự tạo ra các Playlist như "Discovery Weekly" hay "Release Radar" giống hệt Spotify nhưng cho nhạc offline.
*   **Tối ưu cho NAS:** Các tài liệu hướng dẫn về Unraid và Docker Permissions (PUID/PGID) cho thấy tác giả rất quan tâm đến trải nghiệm người dùng tự vận hành server tại nhà.

Đây là một dự án có độ phức tạp cao (~83,000 dòng code Python), giải quyết triệt để bài toán chuyển đổi từ nghe nhạc streaming sang sở hữu nhạc số chất lượng cao (FLAC) một cách tự động.