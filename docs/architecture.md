# Kiến trúc hệ thống — YouTube Auto Publisher

> Trạng thái: **thiết kế đã đối chiếu source thật của 2 upstream** (Phase 0 hoàn tất phần audit).
> Chưa có mã nguồn ứng dụng. Chưa có lần upload thật nào. Chưa test OAuth với tài khoản thật.
>
> Nguồn sự thật cho tài liệu này: `docs/upstream-audit.md` (đọc trực tiếp Go/Python source, không dựa vào README).

---

## 1. Phạm vi

Hệ thống đăng video YouTube tự động, chạy local trên Windows 10/11, nhiều kênh, có dashboard tiếng Việt, hàng đợi bền vững, đặt lịch theo `Asia/Ho_Chi_Minh`.

Kiến trúc **không** tự sản xuất video, không tương tác bình luận, không né quota, không đăng hộ kênh không có quyền.

## 2. Sự thật đã xác minh từ upstream

| Hạng mục | Giá trị thật | Nguồn |
|---|---|---|
| Upload engine | `youtube-uploader-mcp` v0.3.1, commit `985d80f9dc717671155415ba6fde7b362af1c216` | `git describe` = `v0.3.1`, HEAD trùng tag |
| Ngôn ngữ / build | Go 1.24.4, build bằng `go build`; có binary Windows prebuilt `youtube-uploader-mcp-windows-amd64.exe` | `go.mod`, `.github/workflows/release.yaml` |
| Transport | **stdio**, JSON-RPC 2.0, MCP protocol `2025-03-26` (hỗ trợ cả `2024-11-05`) | `main.go` gọi `mcpgo.ServeStdio`; `mark3labs/mcp-go v0.32.0` |
| Số tool thật | **8** — `accesstoken`, `authenticate`, `channels`, `check_job_status`, `list_videos`, `refreshtoken`, `update_video`, `upload_video` | `yum/server.go`, test `tests/mcp_test.go` assert đúng 8 tên và thứ tự |
| Tham số process | `-client_secret_file` (**bắt buộc**), `-working_dir` (mặc định home), `-version` | `main.go` |
| Nơi lưu token | `<working_dir>/.youtube_uploader_channels_cache`, JSON, mode `0600`, **plaintext** | `core/channel.go` |
| Log | `youtube_uploader_mcp.log`, mode `0644`, ghi vào `YOUTUBE_UPLOADER_MCP_LOG_DIR` → CWD → temp → home | `logn/log.go` |
| Scope OAuth xin | `youtube.upload`, `youtube.force-ssl`, `youtube.readonly` | `core/core.go` |
| Upload có early-return | **30 giây** → trả `{"status":"running","job_key":"..."}` | `yum/tool/upload_video.go` |
| Hard timeout upload | **10 phút** (`context.WithTimeout`) → upload dài hơn bị huỷ | `tracker/tracker.go` |
| Tiến độ byte | **Không có** notification/progress trong MCP | không có progress hook trong source |

## 3. Kiến trúc mục tiêu

```text
┌─────────────────────────────┐
│ Browser — React dashboard   │  (tiếng Việt, không giữ token)
└──────────────┬──────────────┘
               │ REST + SSE
┌──────────────▼──────────────┐
│ FastAPI backend             │
│  - API, validation, auth    │
│  - Secret store (DPAPI)     │
└───┬───────────────┬─────────┘
    │               │
    │        ┌──────▼─────────────────────────┐
    │        │ SQLite (SQLAlchemy + Alembic)  │
    │        │ videos / jobs / job_events     │
    │        └──────▲─────────────────────────┘
    │               │ lease + reclaim
┌───▼───────────────────────────────┐
│ Worker tiến trình riêng           │
│  - DB-backed queue, lease, retry  │
│  - scheduler Asia/Ho_Chi_Minh     │
└───┬───────────────────────┬───────┘
    │                       │
┌───▼──────────────────┐  ┌─▼──────────────────────────────┐
│ Media adapter        │  │ YouTube adapter (MCP stdio)    │
│ ffmpeg/ffprobe       │  │ 1 process / worker, không /req │
│ Drive download       │  │ initialize → tools/list → call │
│ ASR + AI metadata    │  │ poll check_job_status          │
└──────────────────────┘  └─┬──────────────────────────────┘
                            │
                   ┌────────▼─────────────────┐
                   │ youtube-uploader-mcp.exe │
                   │  + channels cache file   │
                   └────────┬─────────────────┘
                            │ HTTPS + OAuth2
                   ┌────────▼─────────────────┐
                   │ YouTube Data API v3      │
                   └──────────────────────────┘
```

Nguyên tắc bắt buộc:

1. **Frontend không bao giờ nhận** access token, refresh token, client_secret, API key. Chỉ nhận `channel_id` + tên kênh đã mask.
2. **MCP server không chạy theo từng request.** Một subprocess sống theo vòng đời worker; crash thì restart có backoff.
3. **Không tin `channel_id` do client gửi.** Job lưu `channel_id` ở DB, worker đối chiếu lại trước khi upload.
4. **Không đánh `PUBLISHED`** khi `upload_video` trả về thành công. `upload_video` chỉ chứng minh video đã tồn tại trên YouTube.

## 4. Bảng năng lực: Supported / Needs adapter / Not supported

| Năng lực | Cơ chế | Kết luận | Ghi chú |
|---|---|---|---|
| Upload file local | `upload_video(file_path, ...)` | **Supported** | Path tuyệt đối, cùng máy với MCP process. Không có kiểm tra dung lượng/MIME — ta phải validate trước |
| Title / description / tags / category | `upload_video` | **Supported** | `tags` là **1 string phân tách bằng dấu phẩy**, không phải mảng |
| Ngôn ngữ video | `upload_video(video_language)` | **Supported** | ISO 639-1; set cả `defaultLanguage` + `defaultAudioLanguage` |
| Privacy | `upload_video(status)` | **Supported** | `public`/`unlisted`/`private`, mặc định `private` |
| Schedule lúc upload | `upload_video(publish_at)` | **Supported*** | Ép `private`; *tài khoản phải đủ điều kiện schedule của YouTube |
| Schedule sau upload | `update_video(publish_at)` | **Supported*** | Từ chối nếu video đang `public`; timestamp phải ở tương lai, RFC3339 |
| Đổi privacy sau upload | `update_video(privacy_status)` | **Supported** | |
| Thumbnail | `update_video(thumbnail_path)` | **Supported** | Chặn cứng **>2MB**; cần kênh đã xác minh để dùng thumbnail tuỳ chỉnh |
| Phụ đề | `update_video(subtitle_path, subtitle_language)` | **Supported** | Cần scope `youtube.force-ssl`; `.srt`/`.vtt`; 1 track/lần |
| Thêm vào playlist | `update_video(playlist_id)` | **Supported** | 1 playlist/lần, không tạo playlist |
| Made for kids | cả 2 tool | **Supported** | |
| Synthetic/altered content | cả 2 tool | **Supported** | `contains_synthetic_media` |
| Đa kênh | `authenticate`/`accesstoken`/`channels` | **Needs adapter** | MCP lưu nhiều kênh, nhưng flow OAuth của nó không có state/CSRF đúng nghĩa |
| Refresh token | tự động trong `upload_video`/`update_video`/`list_videos`; hoặc `refreshtoken` | **Needs adapter** | `refreshtoken` trả lỗi nếu `ExpiresIn == 0` (bug nhỏ) |
| Liệt kê video kênh | `list_videos` | **Supported** | Cap 200/lần, catalog cache 60s, tối đa 100 trang × 50 |
| Đối chiếu sau lỗi | `list_videos(query=...)` | **Needs adapter** | Không có API tra theo video ID; phải search theo title rồi so khớp |
| Tiến độ % upload | — | **Not supported** | MCP không phát progress. Chỉ có `running/success/failed` |
| Huỷ upload đang chạy | — | **Not supported** | Không có tool cancel; chỉ huỷ được job còn trong queue |
| Resume upload đứt | — | **Not supported** | Không expose resumable session URI |
| Xoá video | — | **Not supported** | Không có tool delete |
| Sửa title/desc/tags sau upload | — | **Not supported** | `update_video` chỉ đụng `status` + playlist/caption/thumbnail, **không** sửa snippet |
| Đọc/ghi analytics, comment | — | **Not supported** | Ngoài phạm vi |
| Tải từ Google Drive | — | **Needs adapter** | MCP không hỗ trợ; `youtube-publisher` có code tham khảo |
| ASR (Fireworks/Deepgram) | — | **Needs adapter** | Code tham khảo hardcode `ru`, không dùng `--language` |
| Sinh title/description bằng AI | — | **Not supported** | `publish.py` **không** sinh title, không gọi `update_youtube_metadata` |
| Xuất `.srt`/`.vtt` | — | **Not supported** | Upstream chỉ ghi transcript dạng text thường |

## 5. Adapter YouTube (MCP client)

### 5.1 Vòng đời process

```text
spawn(exe, ["-client_secret_file", path, "-working_dir", secretsDir])
  → gửi initialize (protocolVersion "2025-03-26")
  → notifications/initialized
  → tools/list  ← validate đủ 8 tool + schema, sai thì fail-fast
  → dùng cho nhiều job
  → crash/exit → kill sạch, restart backoff 1s/2s/4s… tối đa N lần
```

- Timeout `initialize` 10s, `tools/call` thường 60s, riêng `upload_video` 45s (vì có early-return 30s).
- Mỗi request gắn `id` tăng dần; bỏ qua notification không khớp id; log stderr của server.
- **Không** để MCP ghi log vào thư mục dự án: đặt `YOUTUBE_UPLOADER_MCP_LOG_DIR`.

### 5.2 Upload: bắt buộc poll

`upload_video` trả về **một trong ba** dạng:

```json
{"status": "running", "job_key": "<sha256 hex>", "message": "..."}
```

```json
{"id":"<videoId>","path":"...","title":"...","privacy_status":"private", ...}
```

```text
lỗi → isError=true, text là thông điệp
```

Quy tắc:

1. Nếu `status == "running"` → lưu `job_key` vào DB, chuyển `UPLOADING`, poll `check_job_status(job_key)` mỗi 5–10s.
2. `check_job_status` khi đang chạy trả **plain text** `job is still running, please check in sometime with the same job_key` (không phải JSON) — adapter phải phân biệt.
3. Khi thành công, `check_job_status` trả **nguyên result** của upload (JSON có `id`).
4. `job is not present in registry` ⇒ MCP đã restart, job key mất (in-memory) ⇒ **không upload lại**; chuyển `NEEDS_RECONCILIATION`.

> Đây là điểm khác biệt nghiêm trọng so với plan ban đầu (plan giả định `upload_video` trả video ID ngay).

### 5.3 Idempotency và chống đăng trùng

- `job_key` = `sha256("channel_id" \0 <id> \0 "file_path" \0 <path>)`.
- Trong lúc **đang chạy**, gọi lại cùng cặp (channel, path) trả lại running — an toàn.
- Sau khi **thành công**, gọi lại tạo job mới ⇒ **có thể upload trùng**. Vì vậy tầng ứng dụng phải tự chặn: `idempotency_key` riêng + trạng thái `UPLOADED_*` + fingerprint file.
- Nếu lỗi **sau khi** YouTube đã tạo video: `NEEDS_RECONCILIATION`, dò bằng `list_videos` theo title + khoảng thời gian, đối chiếu trước khi quyết định retry.

### 5.4 Giới hạn cứng phải thiết kế quanh nó

| Giới hạn | Hệ quả thiết kế |
|---|---|
| Hard timeout 10 phút | File lớn + mạng chậm sẽ fail. Phải ước lượng `size / tốc độ upload`; nếu >8 phút thì cảnh báo, hoặc dùng direct API adapter (ADR-001) |
| Không có progress | UI hiển thị tiến độ không xác định + mốc thời gian; không bịa % |
| Registry in-memory | Restart MCP = mất job key ⇒ mọi job `UPLOADING` khi restart phải vào `NEEDS_RECONCILIATION` |
| Không cancel | Nút "huỷ" chỉ có hiệu lực trước khi gọi `upload_video` |
| Không sửa snippet sau upload | Mọi sửa title/desc/tags phải xong **trước** upload |

## 6. Trạng thái video và job

### 6.1 Video

```text
DRAFT → READY → QUEUED → UPLOADING → UPLOADED_PRIVATE → SCHEDULED → PUBLISHED
                                  ↘ FAILED
                                  ↘ NEEDS_RECONCILIATION
```

- `UPLOADED_PRIVATE`: có `youtube_video_id`, privacy thực tế `private`. **Chưa** publish.
- `SCHEDULED`: đã set `publishAt` thành công.
- `PUBLISHED`: chỉ đặt khi đã xác minh `privacyStatus == public` **hoặc** đã qua `publishAt` và đối chiếu trạng thái qua `list_videos`.
- Không có bước nào được nhảy từ `UPLOADING` sang `PUBLISHED`.

### 6.2 Job

```text
PENDING → LEASED → RUNNING → SUCCEEDED
                    ├→ RETRY_WAIT (backoff + jitter, tôn trọng Retry-After)
                    ├→ FAILED (lỗi vĩnh viễn)
                    ├→ NEEDS_RECONCILIATION
                    ├→ REAUTH_REQUIRED
                    └→ CANCELLED
```

- Lease có `lease_owner` + `lease_until`; reclaim job quá hạn khi worker khởi động.
- Mọi chuyển trạng thái ghi `job_events`.

## 7. Hai mốc thời gian (không được lẫn)

| Mốc | Ý nghĩa | Cách làm |
|---|---|---|
| `upload_start_at_utc` | Lúc worker **bắt đầu tải lên** | Lên lịch sớm hơn giờ công bố (ví dụ trước 2–24h) để có thời gian xử lý |
| `publish_at_utc` | Lúc YouTube **công khai** video | Truyền vào `publish_at`, video phải `private` |

Quy tắc:

- Người dùng nhập giờ theo `Asia/Ho_Chi_Minh`; lưu DB bằng UTC; gửi API bằng RFC3339 có offset.
- Nếu `publish_at` ≤ now + 15 phút ⇒ từ chối hoặc gộp về "đăng ngay".
- `publish_at` phải ở tương lai **tại thời điểm gọi API** (`update_video` tự kiểm tra).
- Không dùng `publish_at` cho video đã `public` — MCP sẽ từ chối để tránh unpublish.

## 8. Bảo mật — ranh giới tin cậy

| Thành phần | Giữ gì | Không được giữ |
|---|---|---|
| Browser | session id, dữ liệu hiển thị | token, client_secret, API key |
| FastAPI | session, cấu hình đã mã hoá | (được phép) không log secret |
| SQLite | trạng thái, metadata, `credential_ref` | access/refresh token plaintext |
| MCP process | token YouTube (file `0600`) | API key AI/ASR |
| AI provider | transcript + metadata | token, đường dẫn nhạy cảm |
| Log | thông điệp đã lọc | token, client_secret, full path nhạy cảm |

Điểm yếu đã biết của upstream:

- `AuthURL` dùng `state` cố định `"state-token"` ⇒ **không có CSRF protection**. Ta phải tự sinh nonce, lưu server-side, kiểm tra ở callback (xem `docs/oauth-setup.md`).
- Token lưu plaintext ⇒ thư mục `working_dir` phải được ACL riêng, không nằm trong repo, có hướng dẫn backup.
- Log mode `0644` ⇒ phải đặt log dir riêng và kiểm tra nội dung không chứa token.

## 9. Những gì chưa làm được / cần quyết định

1. **Progress thật** và **upload >10 phút**: cần cân nhắc direct YouTube API adapter. Xem `docs/decisions/ADR-001-mcp-adapter-vs-direct-api.md`.
2. **Sửa metadata sau upload**: hiện không có đường qua MCP; phải chốt metadata trước upload.
3. **Xoá/rollback video**: không có tool; nếu cần thì phải làm tay trên YouTube Studio.
4. **Lưu token**: xem `docs/decisions/ADR-002-token-storage.md`.

## 10. Điều kiện để coi Phase 1 là xong

- [ ] Adapter khởi động được MCP, `tools/list` đủ 8 tool, fail-fast nếu thiếu.
- [ ] Upload 1 video thật ở `private`, lưu được video ID, trả lời đúng khi bị early-return.
- [ ] Poll `check_job_status` xử lý đúng cả 3 dạng kết quả.
- [ ] Restart app: record còn, job `UPLOADING` chuyển `NEEDS_RECONCILIATION`, không upload lại.
- [ ] Không có secret nào xuất hiện trong response API hoặc log.
