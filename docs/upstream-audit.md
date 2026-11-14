# Audit upstream — Phase 0

> Tài liệu này ghi lại **những gì thực sự có trong source**, không phải những gì README nói.
> Mọi kết luận đều kèm đường dẫn file trong `external/`. Hai repo được clone và khóa commit.

---

## 1. Phiên bản đã khóa

| Repo | Commit | Tag | Ngày commit | License |
|---|---|---|---|---|
| `smixs/youtube-publisher` | `fe5555df3740d6796dc35603cc2699821cb3e5ba` | không có tag | 2026-03-06 | MIT (Serge Shima, 2026) |
| `anwerj/youtube-uploader-mcp` | `985d80f9dc717671155415ba6fde7b362af1c216` | **`v0.3.1`** | 2026-11-01 | MIT (anwerj, 2025) |

- Với MCP: `git describe --tags` trả `v0.3.1` và `git rev-list -n1 v0.3.1` **trùng** HEAD ⇒ binary prebuilt `v0.3.1` khớp đúng source đã audit.
- Với publisher: không có tag, chỉ có commit. Muốn nâng cấp phải diff thủ công.
- Commit của MCP **mới hơn** commit của publisher khoảng 8 tháng; hai repo phát triển độc lập, không chia sẻ code.

### Cài đặt trên Windows

```powershell
# Clone (máy này bị lỗi schannel, cần ép openssl — xem docs/troubleshooting.md)
git -c http.sslBackend=openssl clone https://github.com/smixs/youtube-publisher.git external/youtube-publisher
git -c http.sslBackend=openssl clone https://github.com/anwerj/youtube-uploader-mcp.git external/youtube-uploader-mcp

# Ghim đúng commit
git -C external/youtube-publisher checkout fe5555df3740d6796dc35603cc2699821cb3e5ba
git -C external/youtube-uploader-mcp checkout 985d80f9dc717671155415ba6fde7b362af1c216
```

**Không chạy** `scripts/install.ps1` / `install.sh` của upstream: chúng tải binary từ Internet rồi tự sửa `claude_desktop_config.json`, và có `Read-Host` chờ nhập tay. Ta tự tải binary và tự cấu hình.

Binary Windows (nếu không cài Go):

```text
https://github.com/anwerj/youtube-uploader-mcp/releases/download/v0.3.1/youtube-uploader-mcp-windows-amd64.exe
sha256: 5004741158a4d6843f581c59da59516a3d109a9a9ae7acbff27fdab65160ca54
```

Chỉ có `amd64`, **không có** `windows-arm64`.

---

## 2. `youtube-uploader-mcp` — cái thực sự dùng được

### 2.1 Kiến trúc

```text
main.go                  → flag parse, mcpgo.ServeStdio
yum/server.go            → đăng ký 8 tool, hook after-initialize gửi tools/list_changed
yum/tool/*.go            → 8 tool
core/*.go                → OAuth2, YouTube Data API v3, token file, playlist/caption/thumbnail
tracker/*.go             → job registry in-memory + job_key SHA-256
logn/log.go              → log file best-effort
```

- Dùng `mark3labs/mcp-go v0.32.0`, protocol `2025-03-26`.
- Chỉ hỗ trợ **stdio**, không HTTP/SSE.
- `yum.Version` mặc định `0.1.3`; release workflow build **không** truyền `-ldflags -X` ⇒ `initialize.serverInfo.version` sẽ là `0.1.3` bất kể tag `v0.3.1`. Đừng dùng version này để phát hiện phiên bản.

### 2.2 Tool thật (8 tool)

`tools/list` trả theo thứ tự alphabet (test `tests/mcp_test.go` assert đúng):

| Tool | Tham số bắt buộc | Tham số tuỳ chọn | Trả về |
|---|---|---|---|
| `authenticate` | — | `redirect_uri` | URL Google (text) |
| `accesstoken` | `code` | `redirect_uri` | JSON channel đã mask token |
| `channels` | — | — | JSON map channel đã mask |
| `refreshtoken` | `channel_id` | — | JSON channel đã mask |
| `upload_video` | `file_path`, `channel_id`, `description`, `title`, `tags`, `category_id` | `video_language`, `status`, `publish_at`, `made_for_kids`, `contains_synthetic_media` | JSON video **hoặc** JSON running |
| `update_video` | `channel_id`, `video_id` | `playlist_id`, `subtitle_path`, `subtitle_language`, `thumbnail_path`, `publish_at`, `privacy_status`, `contains_synthetic_media`, `made_for_kids` | JSON trạng thái từng hành động |
| `list_videos` | `channel_id` | `query`, `privacy_status`, `order_by`, `direction`, `max`, `offset` | JSON danh sách + phân trang |
| `check_job_status` | `job_key` | — | result gốc, hoặc text "running", hoặc lỗi |

Chi tiết cần lưu ý khi viết adapter:

- `tags` là **string**, server tự `strings.Split(tags, ",")` (`yum/tool/upload_video.go:150`). Không truyền mảng.
- 5 field của `upload_video` được khai báo `Required()`, nhưng server kiểm tra thủ công và trả lỗi text (`upload_video.go:84-87`). Thiếu 1 field = lỗi runtime, không phải lỗi schema — client MCP không tự chặn.
- `status` sai giá trị → lỗi; mặc định `private`.
- `upload_video` kiểm tra file bằng `os.Stat`, từ chối nếu không tồn tại hoặc là thư mục. **Không** kiểm tra phần mở rộng, MIME, dung lượng.
- `update_video` yêu cầu **ít nhất một** tham số hành động, nếu không trả lỗi (`update_video.go:130-134`).
- `update_video` trả **partial success**: mỗi hành động có `*_status` riêng và `errors` map; chỉ trả `isError` khi **tất cả** hành động fail (`update_video.go:302-362`). Adapter phải đọc từng field, không chỉ nhìn `isError`.
- `publish_at` phải là RFC3339 **và ở tương lai**; video đang `public` thì `update_video` từ chối schedule để tránh unpublish (`core/video.go:197-206`).
- `thumbnail_path` bị chặn nếu > 2MB (`update_video.go:171-174`).
- `subtitle_language` mặc định `en`; caption name luôn là `Subtitles (<lang>)`.
- `list_videos` cap `max` ở 200 (`list_videos.go:15`), chỉ sort theo `date`.
- `channels` gọi `ReadChannels(false)` ⇒ **khi chưa từng auth**, tool trả lỗi `failed to get channels: failed to open channels file`.
- `refreshtoken` trả `Invalid access token loaded` nếu `ExpiresIn == 0` (`refresh_token.go:49-51`) — đây là kiểm tra sai, vì token đọc từ file luôn có `ExpiresIn == 0`. Đừng dựa vào tool này; để upload tự refresh.

### 2.3 Cơ chế job bất đồng bộ — điểm dễ sai nhất

```text
upload_video
  ├─ nếu xong trong 30s  → trả JSON video (có id)
  ├─ nếu quá 30s         → trả {"status":"running","job_key":"..."}   ← phải poll
  └─ goroutine tiếp tục, tối đa 10 phút
        └─ quá 10 phút   → job failed: "operation still in progress after 10m0s"
```

- Hằng số: `uploadEarlyReturn = 30s`, `uploadHardTimeout = 10m` (`upload_video.go:16-17`).
- Job registry là `map` **in-memory** trong process (`tracker/tracker.go:34-37`). Restart process ⇒ mất hết `job_key`.
- `check_job_status` khi running trả **plain text**, không JSON (`check_job_status.go:46-47`), trong khi `upload_video` khi running trả **JSON**. Hai định dạng khác nhau cho cùng một khái niệm.
- Job key = SHA-256 của các cặp `field\0value` đã sort, ở đây là `channel_id` + `file_path`.
- Gọi lặp khi đang chạy ⇒ trả lại running, **không** tạo job trùng.
- Sau khi thành công, gọi lại cùng key ⇒ **job mới** ⇒ **upload trùng**. Tầng ứng dụng phải tự chặn.

### 2.4 OAuth và token

```text
core/core.go:41-46   scopes: youtube.upload, youtube.force-ssl, youtube.readonly
core/oauth.go:25     state = "state-token" (hằng số), AccessTypeOffline, ApprovalForce
core/channel.go:13   file .youtube_uploader_channels_cache
core/channel.go:88   os.OpenFile(..., 0600)
```

- `ApprovalForce` + `AccessTypeOffline` ⇒ lần nào cũng trả `refresh_token` (tốt cho đa kênh).
- **`state` là hằng số** ⇒ không chống CSRF. Đây là lỗ hổng của upstream; ta phải bù ở tầng backend.
- Token lưu **plaintext** trong JSON, chỉ bảo vệ bằng mode `0600` (trên Windows mode này gần như không có tác dụng — NTFS ACL mới quyết định).
- Không có cơ chế mã hoá, không có keyring, không có DPAPI.
- `channels`/`accesstoken`/`refreshtoken` đều `Mask()` trước khi trả JSON ⇒ không lộ token ra client MCP (tốt).
- Tự refresh khi token còn < 2 phút trong `upload_video`, `update_video`, `list_videos` và **lưu lại** file. Nếu refresh fail ⇒ lỗi text, không có mã lỗi riêng.

### 2.5 Logging

- File `youtube_uploader_mcp.log`, mode `0644` (`logn/log.go:60`).
- Thứ tự thư mục: `YOUTUBE_UPLOADER_MCP_LOG_DIR` → CWD → OS temp → home. Nếu để mặc định, log rơi vào thư mục dự án.
- Nội dung log chủ yếu là lỗi và tên tool, không log token — nhưng vẫn phải đặt thư mục riêng và kiểm tra.

---

## 3. `youtube-publisher` — chỉ là **tài liệu tham khảo**, không dùng trực tiếp

README hứa pipeline đầy đủ. Source thực tế hẹp hơn nhiều.

### 3.1 Có gì

| Thành phần | File | Đánh giá |
|---|---|---|
| Tải Drive qua REST | `publish.py:96-120` | Dùng được làm tham chiếu; không có resumable, không có progress |
| `ffprobe` lấy duration | `publish.py:123-129` | Dùng được |
| Upload resumable | `publish.py:132-168` | **Không thực sự resumable**: init session rồi `PUT` toàn bộ file một lần, không chia chunk |
| Tách audio 64kbps mono 16kHz | `publish.py:171-181` | Đúng như SKILL mô tả |
| Chia chunk 15 phút | `publish.py:184-198` | Đúng |
| Fireworks Whisper | `publish.py:201-247` | Multipart tự dựng tay |
| Deepgram Nova-3 | `publish.py:250-267` | Đúng |
| Merge transcript + offset | `publish.py:283-311` | Đúng |
| Sinh timestamps | `publish.py:328-358` | Heuristic, **chỉ từ khoá tiếng Nga** |
| OAuth setup CLI | `setup_oauth.py` | Chạy local server port 8914, lưu token plaintext |

### 3.2 Lỗi và thiếu sót đã xác minh

| Vấn đề | Bằng chứng | Ảnh hưởng |
|---|---|---|
| **Không hỗ trợ file local** | Toàn bộ `main()` bắt buộc `url` Drive | Trái với yêu cầu "import file local" của plan |
| **`--language` bị bỏ qua** | Khai báo `publish.py:393`, không dùng; body gửi cứng `"ru"` ở `:223` và URL `language=ru` ở `:258` | Transcript luôn tiếng Nga |
| **`--title` bị bỏ qua** | Khai báo `:395`, không xuất hiện ở bất kỳ đâu khác | Không override được title |
| **Không sinh title/description** | `generate_metadata()` chỉ trả `full_text` + `ts_block` (`:328-358`) | Plan yêu cầu AI sinh 3 phương án — upstream không có |
| **Không cập nhật metadata** | `update_youtube_metadata()` định nghĩa `:361` nhưng **không được gọi** | Video lên YouTube với title `"Processing..."` (`:137-139`) |
| **Nạp cả file vào RAM** | `publish.py:160` `f.read()` | File 2GB ⇒ 2GB RAM, dễ chết máy |
| **Không retry chunk** | `pool.map` không bọc try/except | SKILL nói "retry once" nhưng code không làm |
| **Không kiểm tra quota** | — | Gặp 403 là chết giữa pipeline |
| **Cleanup dùng `/tmp`** | `publish.py:476-480` | Trên Windows `/tmp` không tồn tại; chỉ dọn được `work_dir` |
| **Timestamp tiếng Nga** | `:335` `"Начало"`, `:343-347` từ khoá Nga | Vô dụng cho nội dung Việt/Anh |
| **Mặc định privacy `unlisted`** | `:130`, `:391` | Ngược với yêu cầu "upload private trước" |
| **`json["installed"]` cứng** | `publish.py:70` | Chết nếu client JSON là loại `web` |
| **Không tạo `.srt`/`.vtt`** | `:448-457` chỉ ghi `.txt` | Plan yêu cầu subtitle file |

Kết luận: **không copy `publish.py`**. Dùng nó làm tài liệu tham chiếu cho:
công thức ffmpeg, cách chia chunk, cách merge offset, endpoint Fireworks/Deepgram.

### 3.3 Quy tắc cần sửa nếu dùng công thức của upstream

- `split_audio` dùng `-c copy` sau `-ss` — với mp3 thường ổn nhưng có thể lệch timestamp ở ranh giới chunk. Nên dùng `-ss` **trước** `-i` để seek nhanh và chính xác hơn, hoặc re-encode ở chunk boundary.
- Offset `chunk_index × 900` là xấp xỉ; nếu chunk ngắn hơn 900s (file ngắn) vẫn đúng, nhưng nếu ffmpeg cắt lệch thì transcript lệch dần. Phải dùng duration thực của từng chunk để cộng offset.

---

## 4. Khác biệt so với `YOUTUBE_AUTO_PUBLISHER_PLAN.md`

| Plan nói | Thực tế | Xử lý |
|---|---|---|
| 7 MCP tool | **8** (thêm `check_job_status`) | Bổ sung vào adapter |
| `upload_video` trả video ID ngay | Có thể trả `running` + `job_key` | Bắt buộc poll |
| "cập nhật thumbnail/subtitle/playlist" | `update_video` làm được, nhưng **không sửa snippet** | Chốt metadata trước upload |
| "upload resumable" (publisher) | Chỉ là 1 PUT toàn bộ | Không dựa vào |
| "youtube-publisher tạo title/description" | Không hề | Tự làm AI adapter |
| "6 video/ngày" | Là suy ra từ quota, không phải hằng số API | Không hardcode; hiển thị cảnh báo |
| Direct API adapter "chỉ khi cần" | Cần cho progress + upload >10 phút | Xem ADR-001 |

---

## 5. Attribution và cập nhật upstream

### Attribution

- `youtube-uploader-mcp` — MIT, Copyright (c) 2025 anwerj. Nếu phân phối lại binary/source, phải giữ nguyên LICENSE và copyright notice.
- `youtube-publisher` — MIT, Copyright (c) 2026 Serge Shima. Nếu copy đoạn code (kể cả công thức ffmpeg), giữ notice.
- Không có đoạn code nào của upstream được copy vào dự án ở thời điểm này.

### Chính sách cập nhật

1. `external/` **gitignored** — không commit upstream vào repo chính.
2. Ghi commit đang dùng vào `docs/upstream-audit.md` (file này) và vào `.env.example` dạng comment.
3. Khi nâng cấp: `git fetch` → `git diff <old> <new> -- '*.go'` → đọc diff của `yum/tool/`, `core/`, `tracker/` → cập nhật lại bảng 2.2 và mục 4.
4. Test lại bằng mock trước, sau đó mới chạy thật.
5. Không tự động cập nhật binary lúc runtime.

---

## 6. Việc chưa kiểm chứng (phải nói rõ)

- **Chưa chạy** binary MCP lần nào trên máy này (chưa tải).
- **Chưa** thực hiện OAuth với tài khoản Google thật.
- **Chưa** upload video nào.
- **Chưa** kiểm chứng quota hiện tại của YouTube Data API v3 — phải đọc tài liệu chính thức tại thời điểm triển khai.
- Các nhận định về hành vi runtime (early-return, 10 phút, token file) là **đọc từ source**, không phải quan sát thực nghiệm.
