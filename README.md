# DangVideoTuDong — YouTube Auto Publisher

Hệ thống đăng video YouTube tự động đa kênh, chạy local trên Windows, dashboard tiếng Việt, hàng đợi bền vững và đặt lịch theo giờ Việt Nam.

> **Trạng thái hiện tại: chỉ có tài liệu. Chưa có mã nguồn ứng dụng.**
> Chưa upload video nào. Chưa test OAuth với tài khoản thật. Chưa chạy binary MCP.

---

## Repository này có gì

```text
.
├── YOUTUBE_AUTO_PUBLISHER_PLAN.md     # Đặc tả gốc (không sửa)
├── docs/
│   ├── architecture.md                # Kiến trúc + bảng năng lực + giới hạn
│   ├── upstream-audit.md              # Audit source 2 repo, commit, bug, license
│   ├── oauth-setup.md                 # Google Cloud, scope, luồng callback, đa kênh
│   ├── data-model.md                  # Schema DB, trạng thái, idempotency
│   ├── troubleshooting.md             # Lỗi thật + cách sửa
│   ├── roadmap.md                     # Phase 0–4 và gate
│   └── decisions/
│       ├── ADR-001-mcp-adapter-vs-direct-api.md
│       └── ADR-002-token-storage.md
└── external/                          # 2 repo upstream (không commit)
    ├── youtube-publisher/             # fe5555d
    └── youtube-uploader-mcp/          # v0.3.1 / 985d80f
```

## Đọc theo thứ tự

1. `YOUTUBE_AUTO_PUBLISHER_PLAN.md` — yêu cầu gốc.
2. `docs/upstream-audit.md` — cái gì **thực sự** có trong 2 repo (khác README của họ ở đâu).
3. `docs/architecture.md` — thiết kế và giới hạn phải sống chung.
4. `docs/oauth-setup.md` — trước khi bấm "Kết nối kênh".
5. `docs/roadmap.md` — làm gì tiếp.

## Sự thật quan trọng nhất

Ba điều dưới đây khác với giả định trong plan gốc, và thay đổi cách thiết kế:

1. **`upload_video` không trả video ID ngay.** Sau 30 giây nó trả `{"status":"running","job_key":"..."}` và bạn phải poll `check_job_status`.
2. **Upload bị huỷ cứng sau 10 phút.** Video lớn hoặc mạng chậm sẽ fail; phải ước lượng trước.
3. **MCP không có progress %, không cancel, không sửa title/description sau upload.** Mọi metadata phải chốt trước khi upload.

Ngoài ra: MCP lưu token YouTube **plaintext** trong file, và luồng OAuth của nó dùng `state` cố định (không chống CSRF) — backend phải tự bù.

## Upstream đã ghim

| Repo | Commit | Ghi chú |
|---|---|---|
| `smixs/youtube-publisher` | `fe5555df3740d6796dc35603cc2699821cb3e5ba` | MIT, chỉ dùng làm tham chiếu |
| `anwerj/youtube-uploader-mcp` | `985d80f9dc717671155415ba6fde7b362af1c216` = tag `v0.3.1` | MIT, binary Windows prebuilt khớp source |

```powershell
git -c http.sslBackend=openssl clone https://github.com/smixs/youtube-publisher.git external/youtube-publisher
git -c http.sslBackend=openssl clone https://github.com/anwerj/youtube-uploader-mcp.git external/youtube-uploader-mcp
git -C external/youtube-publisher checkout fe5555df3740d6796dc35603cc2699821cb3e5ba
git -C external/youtube-uploader-mcp checkout 985d80f9dc717671155415ba6fde7b362af1c216
```

## Môi trường máy hiện tại

| Công cụ | Trạng thái |
|---|---|
| Python 3.12.0 | có |
| Node 24.16.0 / npm 11.13.0 | có |
| git | có (cần `http.sslBackend=openssl` để clone) |
| Go | **chưa có** — dùng binary prebuilt, không build từ source |
| ffmpeg / ffprobe | **chưa có** — cần cho Phase 3 |
| Google OAuth client | **chưa có** |
| API key AI/ASR | **chưa có** |

## Bước tiếp theo

Theo `docs/roadmap.md`, Phase 1 bắt đầu bằng:

1. Scaffold backend + frontend + migration.
2. `YouTubeMcpClient` với `tools/list` validation.
3. OAuth có nonce one-time.
4. Import file local + upload private thật.

Trước đó cần người dùng cung cấp:

- `client_secret.json` (Desktop app) của Google Cloud project đã bật YouTube Data API v3.
- Xác nhận kênh YouTube nào dùng để test (và email nằm trong Test users).
- Quyết định có dùng Google Drive ngay ở Phase 1 hay để Phase 3.

## Giấy phép & ghi công

Code dự án: chưa có. Tài liệu: nội bộ.

Upstream (MIT):

- `youtube-uploader-mcp` — Copyright (c) 2025 anwerj.
- `youtube-publisher` — Copyright (c) 2026 Serge Shima.

Giữ nguyên LICENSE tương ứng nếu phân phối lại hoặc copy code.
