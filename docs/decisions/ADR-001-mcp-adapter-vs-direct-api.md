# ADR-001 — Dùng MCP stdio làm upload engine, chưa viết direct YouTube API adapter

- **Ngày**: 2026-11-14
- **Trạng thái**: Accepted (có điều kiện xem lại)
- **Bối cảnh**: Phase 0 audit

## Bối cảnh

Plan yêu cầu dùng `youtube-uploader-mcp` làm upload engine qua MCP, chỉ thêm direct API adapter nếu MCP không đáp ứng. Sau khi đọc source, MCP có 4 giới hạn ảnh hưởng trực tiếp tới yêu cầu sản phẩm:

1. **Không có progress %** — không hook, không notification. Chỉ `running/success/failed`.
2. **Hard timeout 10 phút** cho `upload_video` — quá hạn thì upload bị huỷ giữa đường.
3. **Không cancel** upload đang chạy.
4. **Không sửa được snippet** (title/description/tags) sau upload — `update_video` chỉ đụng `status`, playlist, caption, thumbnail.

Trong khi đó plan yêu cầu "báo tiến độ", "hủy job", và cho phép sửa metadata trước khi đăng.

## Quyết định

**Phase 1–2 dùng MCP adapter làm engine duy nhất.** Không viết direct API adapter ngay.

Bù lại ở tầng ứng dụng:

- Progress: hiển thị **trạng thái + mốc thời gian + ước lượng**, tuyệt đối không bịa %.
- Upload > 10 phút: ước lượng trước khi chạy (`size / tốc độ đo được`), nếu > 8 phút thì cảnh báo người dùng và đề nghị nén/chia nhỏ; ghi lỗi vào `job_events` nếu vượt.
- Cancel: chỉ cho huỷ job ở `PENDING`/`RETRY_WAIT`; nút huỷ bị disable khi `UPLOADING`, kèm giải thích.
- Metadata: khoá form sau khi bắt đầu upload; mọi sửa đổi phải hoàn tất trước khi job chạy.

Định nghĩa interface `YouTubeProvider` ngay từ đầu (upload, poll, update, list, refresh, health) để direct adapter có thể thêm sau mà không đổi tầng nghiệp vụ.

## Hệ quả

### Tích cực

- Không phải tự viết OAuth2, resumable upload, caption/thumbnail/playlist — giảm rủi ro và thời gian Phase 1.
- Multi-channel và refresh token đã có sẵn.
- Ít code phải bảo trì; upstream MIT, commit đã ghim và trùng tag release ⇒ binary prebuilt dùng được, không cần Go.
- Khớp với plan gốc, dễ giải trình.

### Tiêu cực / rủi ro chấp nhận

- **Không có progress thật** ⇒ UX kém hơn kỳ vọng ở mục "báo tiến độ".
- **Video lớn + mạng chậm = fail chắc chắn** ở mốc 10 phút. Với 10 Mbps, trần thực tế khoảng 700–750 MB.
- **Không rollback**: upload sai metadata thì phải xoá/sửa tay trên YouTube Studio.
- **Registry in-memory** ⇒ restart MCP làm mất `job_key`, buộc phải reconciliation.
- Phụ thuộc vào một binary bên thứ ba, không có version API ổn định; breaking change có thể xảy ra ở minor version.

### Trung tính

- Token YouTube do MCP giữ (xem ADR-002), không do DB của ta.

## Điều kiện xem lại (gate)

Thêm direct API adapter khi **bất kỳ** điều nào đúng:

1. Người dùng thực tế có video > 750 MB hoặc đường upload < 15 Mbps.
2. Yêu cầu progress % trở thành bắt buộc (không chỉ "nice to have").
3. Cần sửa snippet sau upload, hoặc cần xoá video/rollback.
4. Cần resume upload đứt kết nối.
5. Upstream ngừng bảo trì hoặc ra bản phá vỡ tương thích mà không pin được.

Khi đó: viết `DirectYouTubeProvider` dùng `google-api-python-client`, chạy song song và chọn theo cấu hình `YOUTUBE_PROVIDER=mcp|direct`. Không xoá MCP adapter.

## Phương án đã cân nhắc và loại

| Phương án | Lý do loại |
|---|---|
| Gọi `publish.py` của `youtube-publisher` | Không hỗ trợ file local; bỏ qua `--language`, `--title`; không sinh metadata; nạp cả file vào RAM; không retry. Sửa nó tốn hơn viết mới |
| Viết direct API ngay từ đầu | Tăng khối lượng Phase 1 (OAuth, resumable, caption, thumbnail, playlist) trong khi MCP đã có sẵn và đã audit |
| Fork MCP và sửa timeout thành 60 phút | Có thể làm sau; hiện chưa cần vì chưa có video lớn thực tế. Ghi lại như lối thoát nhanh nếu cần |
| Dùng `yt-dlp`/`youtubeuploader` CLI | Không có caption/thumbnail/playlist đầy đủ, không có multi-channel token model |

## Ghi chú triển khai

- Adapter phải **fail-fast** nếu `tools/list` không đủ 8 tool — tránh chạy với binary cũ thiếu `check_job_status`.
- Log rõ `serverInfo.version` (sẽ luôn là `0.1.3` do release build không truyền ldflags) và SHA-256 của binary để truy vết.
