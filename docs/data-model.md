# Thiết kế dữ liệu tối thiểu

> SQLite + SQLAlchemy + Alembic. Mọi cột thời gian lưu **UTC**; hiển thị theo `Asia/Ho_Chi_Minh`.
> Không có cột nào lưu access/refresh token YouTube (xem ADR-002).

---

## 1. Bảng

### `channels`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | integer PK | |
| `youtube_channel_id` | text UNIQUE NOT NULL | `UC...` từ Google, không phải email |
| `display_name` | text NOT NULL | |
| `custom_url` | text NULL | |
| `credential_ref` | text NOT NULL | đường dẫn `working_dir` chứa token của MCP |
| `status` | text NOT NULL | `CONNECTED` / `REAUTH_REQUIRED` / `DISCONNECTED` |
| `last_verified_at` | datetime NULL | lần cuối `channels` xác nhận còn hợp lệ |
| `created_at` / `updated_at` | datetime | |

Bất biến: không có 2 hàng cùng `youtube_channel_id`. Xoá kênh = `DISCONNECTED`, **không** xoá cứng (giữ lịch sử job).

### `videos`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | integer PK | |
| `source_type` | text | `LOCAL` / `DRIVE` |
| `source_uri` | text | local: **đường dẫn tương đối trong staging**; Drive: file ID + URL gốc |
| `fingerprint` | text | SHA-256 nội dung (hoặc hash phần đầu+cỡ nếu tối ưu) |
| `file_size` | bigint | byte |
| `duration_sec` | real | từ `ffprobe` |
| `width` / `height` / `codec`/ `container` | | từ `ffprobe`, phục vụ validate |
| `title` | text | ≤ 100 ký tự, validate trước upload |
| `description` | text | ≤ 5000 ký tự |
| `tags_json` | text | mảng JSON; adapter join `,` khi gọi MCP |
| `category_id` | text | mặc định `28` |
| `language` | text | ISO 639-1 |
| `audience_made_for_kids` | boolean | bắt buộc chọn trước upload |
| `synthetic_content_flag` | boolean NULL | NULL = không khai báo |
| `privacy` | text | `private` / `unlisted` / `public` (giá trị mong muốn cuối cùng) |
| `thumbnail_path` | text NULL | trong `storage/thumbnails` |
| `transcript_path` | text NULL | |
| `subtitle_path` | text NULL | `.srt`/`.vtt` |
| `subtitle_language` | text NULL | |
| `target_channel_id` | integer FK → channels | **bắt buộc**, không suy diễn |
| `youtube_video_id` | text NULL | chỉ có sau khi upload thành công |
| `youtube_upload_status` | text NULL | `private`/`unlisted`/`public` **thực tế** trên YouTube |
| `state` | text | xem §2 |
| `created_at` / `updated_at` | datetime | |

Index: `(fingerprint)`, `(state)`, `(target_channel_id)`, `(youtube_video_id)` UNIQUE khi NOT NULL.

### `jobs`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | integer PK | |
| `video_id` | FK → videos | |
| `channel_id` | FK → channels | kiểm tra lại ở worker |
| `kind` | text | `UPLOAD` / `PUBLISH_UPDATE` / `RECONCILE` |
| `status` | text | xem §3 |
| `priority` | integer | nhỏ hơn = trước |
| `scheduled_upload_at_utc` | datetime | **upload time** |
| `publish_at_utc` | datetime NULL | giờ công khai; NULL = đăng ngay sau upload |
| `attempts` / `max_attempts` | integer | mặc định 5 |
| `next_run_at_utc` | datetime | backoff |
| `idempotency_key` | text UNIQUE | `sha256(video_id|channel_id|attempt_group)` |
| `mcp_job_key` | text NULL | `job_key` do `upload_video` trả về khi early-return |
| `lease_owner` | text NULL | worker id |
| `lease_until` | datetime NULL | reclaim nếu quá hạn |
| `last_error_code` | text NULL | mã chuẩn hoá của ta, không phải text của MCP |
| `last_error_detail` | text NULL | đã redact |
| `created_at` / `updated_at` | datetime | |

Index: `(status, next_run_at_utc)`, `(lease_until)`, `(video_id)`.

### `job_events`

| Cột | Kiểu |
|---|---|
| `id` | integer PK |
| `job_id` | FK → jobs |
| `level` | `info` / `warn` / `error` |
| `event_type` | text (`TOOL_CALL`, `EARLY_RETURN`, `POLL`, `RETRY`, `RECONCILE`, …) |
| `message_redacted` | text |
| `created_at` | datetime |

Không lưu payload thô của tool call (có thể chứa path). Chỉ lưu tóm tắt.

### `settings`

| Cột | Kiểu |
|---|---|
| `key` | text PK |
| `value_encrypted` | text NULL (base64 blob DPAPI) |
| `value_plain` | text NULL (chỉ cho giá trị không nhạy cảm) |
| `is_secret` | boolean |
| `updated_at` | datetime |

Bất biến: `is_secret = true` ⇒ chỉ được ghi `value_encrypted`.

### `ai_generations`

| Cột | Kiểu |
|---|---|
| `id` | integer PK |
| `video_id` | FK |
| `provider` | text |
| `prompt_version` | text |
| `output_json` | text |
| `cost_estimate` | real NULL |
| `created_at` | datetime |

### `oauth_states` (bổ sung so với plan)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `nonce` | text PK | 32 byte hex |
| `created_at` | datetime | |
| `expires_at` | datetime | now + 10 phút |
| `used_at` | datetime NULL | one-time |
| `session_id` | text | ràng buộc với phiên trình duyệt |

Lý do: MCP dùng `state` cố định ⇒ backend phải tự chống CSRF. Không có bảng này thì callback dễ bị replay.

---

## 2. Trạng thái video

```text
DRAFT
  → READY            (đã có file hợp lệ + metadata tối thiểu)
  → QUEUED           (đã tạo job)
  → UPLOADING
  → UPLOADED_PRIVATE (có youtube_video_id, privacy thực tế = private)
  → SCHEDULED        (đã set publishAt thành công)
  → PUBLISHED        (đã xác minh public)
```

Nhánh lỗi:

```text
bất kỳ → FAILED
bất kỳ → NEEDS_RECONCILIATION   (nghi ngờ YouTube đã tạo video)
bất kỳ → REAUTH_REQUIRED        (token kênh hỏng)
QUEUED  → CANCELLED             (chỉ trước khi UPLOADING)
```

Quy tắc tuyệt đối:

1. `PUBLISHED` **chỉ** sau khi đọc lại trạng thái thật, không suy ra từ việc upload trả 200.
2. `UPLOADED_PRIVATE` là trạng thái **bắt buộc** đi qua nếu có `publish_at`.
3. Không cho `UPLOADING → PUBLISHED`.

## 3. Trạng thái job

```text
PENDING → LEASED → RUNNING → SUCCEEDED
                    ├→ RETRY_WAIT
                    ├→ FAILED
                    ├→ NEEDS_RECONCILIATION
                    ├→ REAUTH_REQUIRED
                    └→ CANCELLED
```

- `LEASE` có thời hạn (ví dụ 15 phút) và được gia hạn khi job còn chạy.
- Worker khởi động: mọi job `LEASED/RUNNING` có `lease_until < now` → `RETRY_WAIT` **hoặc** `NEEDS_RECONCILIATION` nếu đã từng gọi `upload_video` (xem `mcp_job_key`).

## 4. Idempotency và chống trùng

- `idempotency_key` sinh một lần khi tạo job; retry **không** đổi key.
- Trước khi gọi `upload_video`, kiểm tra:
  1. `videos.youtube_video_id IS NULL`
  2. `videos.state = UPLOADING`
  3. `jobs.mcp_job_key` rỗng hoặc job cũ đã kết thúc
- Nếu `mcp_job_key` khác rỗng ⇒ **chỉ poll**, không gọi upload lần nữa.
- Nếu `check_job_status` trả `job is not present in registry` ⇒ `NEEDS_RECONCILIATION`, dò bằng `list_videos(query=title)`, so khớp title + thời gian tạo, **không** upload lại.

## 5. Thời gian

- DB luôn UTC (`datetime` naive UTC hoặc ISO có `Z`).
- Input từ UI: `"2026-11-20 20:00"` + timezone `Asia/Ho_Chi_Minh` → convert bằng `zoneinfo`.
- Gửi MCP: RFC3339 **có offset** hoặc `Z`; `update_video` từ chối nếu không ở tương lai.
- Không dùng `datetime.now()` không tz. Không dùng `utcnow()` (deprecated ở Python 3.12).
- `scheduled_upload_at_utc` phải `< publish_at_utc` nếu có; nếu người dùng chỉ nhập giờ công bố, mặc định upload trước 2 giờ (cấu hình được), tối thiểu 15 phút.

## 6. Retention

| Dữ liệu | Giữ | Ghi chú |
|---|---|---|
| Video gốc trong `staging/` | tới khi `PUBLISHED` + 7 ngày | cấu hình được |
| Audio/transcript tạm | 24 giờ | xoá sau khi tạo subtitle |
| `job_events` | 90 ngày | |
| `ai_generations` | 90 ngày | |
| Log file | 30 ngày, rotate | không chứa secret |
| `oauth_states` đã dùng | 24 giờ | dọn định kỳ |
