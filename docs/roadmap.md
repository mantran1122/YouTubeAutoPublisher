# Roadmap & trạng thái

> Cập nhật: 2026-11-14. Đây là bản đồ thực thi, không phải danh sách mong muốn.
> Nguyên tắc: mỗi phase phải **chạy được**, có test, rồi mới sang phase sau.

---

## Tổng quan

| Phase | Nội dung | Trạng thái |
|---|---|---|
| 0 | Audit upstream + tài liệu kiến trúc | **Hoàn tất phần audit/tài liệu** — không có code |
| 1 | MVP: 1 kênh, import local, upload private | Chưa bắt đầu |
| 2 | Queue bền vững, lịch, retry, đa kênh | Chưa bắt đầu |
| 3 | Drive, ASR, AI metadata, phụ đề | Chưa bắt đầu |
| 4 | E2E, bảo mật, README vận hành | Chưa bắt đầu |

---

## Phase 0 — Audit (đã xong)

Đã làm:

- [x] Clone 2 repo, ghim commit, xác minh tag.
- [x] Đọc `SKILL.md`, `README.md`, toàn bộ Go source của MCP, `publish.py`, `setup_oauth.py`, license.
- [x] Xác định **8** tool thật, schema, định dạng trả về (kể cả bất đối xứng JSON/text).
- [x] Phát hiện early-return 30s, hard timeout 10 phút, registry in-memory.
- [x] Phát hiện token lưu plaintext, `state` cố định trong OAuth.
- [x] Phát hiện `publish.py` bỏ qua `--language`/`--title`, không sinh metadata, không hỗ trợ file local.
- [x] Viết `docs/architecture.md`, `docs/upstream-audit.md`, `docs/oauth-setup.md`, `docs/troubleshooting.md`, `docs/data-model.md`, 2 ADR.

Chưa làm (cần code để làm):

- [ ] Chạy thử binary MCP, chụp lại `initialize` + `tools/list` thật.
- [ ] OAuth với tài khoản thật.
- [ ] Đo tốc độ upload thực tế để hiệu chỉnh ngưỡng 8 phút.

## Phase 1 — MVP

Mục tiêu: một người dùng, một kênh, upload thật một video private từ file local.

- [ ] Scaffold backend FastAPI + SQLite + Alembic; `GET /api/health`.
- [ ] Scaffold frontend React + Vite + Tailwind, tiếng Việt, dark mode.
- [ ] `YouTubeMcpClient`: spawn, initialize, validate 8 tool, call, timeout, restart.
- [ ] OAuth flow với `oauth_states` (nonce one-time).
- [ ] Bảng `channels`, hiển thị kênh + trạng thái token.
- [ ] Import file local: chọn/kéo thả, validate extension + `ffprobe`, fingerprint.
- [ ] Form metadata: title/description/tags/category/language/privacy/audience.
- [ ] Upload thủ công `private`, xử lý early-return + poll.
- [ ] Dashboard trạng thái thật, không dữ liệu giả.
- [ ] Restart app: record còn, job dở dang vào `NEEDS_RECONCILIATION`.

**Gate**: upload thành công 1 video private thật, có video ID, không lộ secret trong response/log.

## Phase 2 — Automation core

- [ ] Worker tiến trình riêng + DB-backed queue (lease, reclaim).
- [ ] Lịch: `scheduled_upload_at_utc` vs `publish_at_utc`, timezone VN.
- [ ] Retry backoff + jitter, tôn trọng `Retry-After`, phân loại lỗi vĩnh viễn/tạm thời.
- [ ] `NEEDS_RECONCILIATION` + dò bằng `list_videos`.
- [ ] `update_video`: thumbnail, subtitle, playlist, privacy, audience, synthetic.
- [ ] Đa kênh: kiểm tra `job.channel_id` khớp kênh trước mọi lần upload.
- [ ] SSE tiến độ (trạng thái, không bịa %).
- [ ] Trang lịch + lịch sử + logs.

**Gate**: 5 video trong queue, restart giữa chừng không mất job, không upload trùng.

## Phase 3 — AI & Drive

- [ ] Drive download (scope `drive.readonly`, client riêng), streaming, giới hạn dung lượng.
- [ ] ffmpeg: tách audio, chia chunk theo duration thật, offset chính xác.
- [ ] ASR adapter Fireworks/Deepgram — **không hardcode `ru`**.
- [ ] Sinh `.srt`/`.vtt` từ segment.
- [ ] AI metadata: 3 phương án title, description, tags, chapters; kiểm tra chapter ≤ duration.
- [ ] Fallback hoàn toàn thủ công khi không có API key.
- [ ] Kiểm soát chi phí: giới hạn phút/phút, cache theo fingerprint.

**Gate**: tắt hết AI key vẫn upload được; bật key thì có transcript + 3 phương án.

## Phase 4 — Hoàn thiện

- [ ] pytest: API, timezone, early-return, poll 3 dạng, retry, reclaim, dedupe.
- [ ] Mock MCP server cho CI (JSON-RPC qua stdio).
- [ ] Playwright smoke: import → edit → schedule → queue → complete.
- [ ] Rà soát bảo mật: secret trong log, path traversal, SSRF, CORS/CSRF, upload validation.
- [ ] `setup_windows.ps1`, `start_windows.bat`, `.env.example`.
- [ ] README vận hành: backup, cleanup, re-auth, đọc log.
- [ ] `docs/test-report.md` với kết quả thật.

**Gate**: checklist nghiệm thu ở `YOUTUBE_AUTO_PUBLISHER_PLAN.md` §10 được tick trung thực.

---

## Rủi ro theo dõi

| Rủi ro | Phase | Đối sách |
|---|---|---|
| Video lớn vượt 10 phút | 1–2 | Ước lượng trước; ADR-001 gate |
| Không có progress % | 2 | UI trạng thái + mốc thời gian |
| Token plaintext | 1 | ACL thư mục; ADR-002 |
| OAuth thiếu CSRF | 1 | `oauth_states` nonce |
| Refresh token 7 ngày (Testing) | 1 | Publish consent screen |
| Quota YouTube | 2 | Phân loại lỗi, retry sang ngày |
| `ffmpeg` chưa cài | 3 | Preflight check + thông báo |

## Những gì sẽ **không** làm

- Tự sản xuất video, tự comment, tương tác giả.
- Né quota, xoay nhiều tài khoản để lách giới hạn.
- Upload video không có quyền.
- Đánh dấu `PUBLISHED` khi chưa xác minh.
- Tuyên bố "đã test OAuth/upload thật" khi chưa có bằng chứng.
