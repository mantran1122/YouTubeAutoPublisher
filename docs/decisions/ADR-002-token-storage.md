# ADR-002 — Ranh giới lưu token và secret

- **Ngày**: 2026-11-14
- **Trạng thái**: Accepted
- **Bối cảnh**: Plan cấm lưu access/refresh token plaintext trong DB/log, yêu cầu DPAPI hoặc credential store của OS.

## Bối cảnh

Có hai nhóm secret khác nhau, không thể xử lý giống nhau:

| Nhóm | Ai tạo | Ai dùng | Đặc điểm |
|---|---|---|---|
| YouTube access/refresh token | MCP `accesstoken` | chỉ MCP | MCP tự lưu file `.youtube_uploader_channels_cache`, plaintext, mode 0600 |
| Google OAuth client_secret | người dùng tải từ Cloud Console | MCP (đọc file) + backend (nếu Drive) | file JSON, MCP nhận qua `-client_secret_file` |
| API key AI/ASR (Fireworks, Deepgram, LLM) | người dùng | backend | cấu hình qua UI, phải mã hoá |
| Session secret, CSRF nonce | backend | backend | sinh runtime, không cần lưu lâu |

MCP **không** cung cấp API để đọc/ghi token có kiểm soát; nó chỉ đọc/ghi file nội bộ. Việc ta ghi trực tiếp vào file đó sẽ phụ thuộc định dạng nội bộ (`oauth2.Token` JSON) — dễ vỡ khi upstream đổi.

## Quyết định

1. **YouTube token thuộc sở hữu của MCP.** DB của ta **không** lưu token. Chỉ lưu `credential_ref` = đường dẫn thư mục `working_dir` + `youtube_channel_id`.
2. **Không đọc, không parse, không ghi** `.youtube_uploader_channels_cache` từ code ứng dụng. Mọi thao tác token đi qua MCP tool.
3. **Cô lập thư mục token**: `%LOCALAPPDATA%\DangVideoTuDong\secrets\` — ngoài repo, ACL chỉ user hiện tại + SYSTEM. Không đặt trong `storage/` (dễ bị backup/zip chia sẻ).
4. **Secret của app** (AI key, Drive client secret copy, session key) mã hoá bằng **Windows DPAPI** (`CryptProtectData`, scope CurrentUser) qua package `pywin32`/`ctypes`; lưu blob base64 trong bảng `settings`. Trên Linux: dùng `keyring` nếu có, fallback file `0600` + cảnh báo rõ trong README.
5. **Không bao giờ** trả secret qua API. Endpoint settings chỉ trả `configured: true/false` + 4 ký tự cuối.
6. **Log**: mọi log đi qua bộ lọc redaction (regex token, `Bearer`, `client_secret`, `refresh_token`, API key pattern). MCP log nằm thư mục riêng, không nằm trong repo.
7. **Backup**: chỉ backup `storage/app.db` + `storage/`. Token YouTube **không** backup được sang máy khác — phải auth lại (DPAPI gắn với user/máy). Ghi rõ trong README.

## Hệ quả

### Tích cực

- Không phụ thuộc định dạng file nội bộ của upstream ⇒ nâng cấp MCP ít rủi ro.
- DB không chứa token ⇒ rò rỉ DB (backup, sync cloud) không lộ kênh YouTube.
- App secret được mã hoá thật bằng OS, không chỉ "ẩn trong UI".

### Tiêu cực

- Token YouTube vẫn **plaintext trên đĩa** (do MCP quyết định). Ta chỉ giảm thiểu bằng ACL + vị trí. Phải nói rõ trong README, không được tuyên bố "token được mã hoá".
- Restore sang máy khác = auth lại toàn bộ kênh.
- DPAPI hỏng (đổi user profile, roam profile) ⇒ mất app secret, phải nhập lại AI key.
- Nếu người dùng copy thư mục `secrets/` đi nơi khác, token vẫn đi theo — cần cảnh báo khi export.

### Rủi ro còn lại

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Malware đọc file token dưới cùng user | Cao (không tránh được với app desktop) | ACL, cảnh báo, không lưu trong repo/sync folder |
| Log MCP ghi vào CWD dự án | Trung bình | Set `YOUTUBE_UPLOADER_MCP_LOG_DIR` |
| Người dùng commit `client_secret.json` | Cao | `.gitignore` + pre-commit check + hướng dẫn |
| Token lộ qua prompt AI | Trung bình | AI chỉ nhận transcript/metadata; không bao giờ nhận token/đường dẫn tuyệt đối |

## Phương án đã loại

| Phương án | Lý do |
|---|---|
| Tự ghi token đã mã hoá vào `.youtube_uploader_channels_cache` | Phụ thuộc định dạng nội bộ; MCP không hỗ trợ; dễ vỡ khi nâng cấp |
| Fork MCP để thêm DPAPI | Tăng gánh nặng bảo trì; chỉ nên làm nếu có yêu cầu bảo mật cao hơn |
| Lưu token trong DB của ta | Vi phạm trực tiếp yêu cầu của plan |
| Dùng `keyring` cho mọi thứ trên Windows | `keyring` dùng Credential Manager — tốt, nhưng thêm dependency; DPAPI đủ cho blob settings |
