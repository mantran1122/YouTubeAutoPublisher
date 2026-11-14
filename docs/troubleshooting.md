# Troubleshooting

> Mọi mục dưới đây đều xuất phát từ source đã đọc hoặc lỗi đã gặp thật trên máy Windows này.
> Mục nào chưa kiểm chứng sẽ ghi rõ.

---

## A. Cài đặt / môi trường

### A1. `git clone` lỗi `schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS`

**Đã gặp thật** khi clone 2 repo upstream trên máy này.

```text
fatal: unable to access 'https://github.com/...': schannel: AcquireCredentialsHandle
failed: SEC_E_NO_CREDENTIALS (0x8009030e) - No credentials are available in the security package
```

Nguyên nhân: Git for Windows dùng backend schannel, không lấy được credential trong phiên không tương tác.

Cách sửa:

```powershell
git -c http.sslBackend=openssl clone <url>
# hoặc cố định cho repo hiện tại:
git config --local http.sslBackend openssl
```

Nếu vẫn lỗi, kiểm tra `git config --global --get http.sslBackend` và biến `GIT_SSL_NO_VERIFY` (đừng bật vô tội vạ).

### A2. Không có `go` trên máy

`go` không có trong PATH ⇒ **không build từ source**. Dùng binary prebuilt `v0.3.1` (release tương ứng đúng commit đã audit, xem `docs/upstream-audit.md`).

Nếu buộc phải build: cần **Go 1.24.4+** (`go.mod`), và internet để tải module.

### A3. Không có `ffmpeg`/`ffprobe`

**Đã kiểm tra: máy này chưa có.** Ảnh hưởng Phase 3 (tách audio, đo duration, tạo thumbnail). Upload thủ công không cần ffmpeg, nhưng validate file và transcript thì cần.

```powershell
winget install Gyan.FFmpeg
# hoặc tải từ https://www.gyan.dev/ffmpeg/builds/ rồi thêm bin vào PATH
ffmpeg -version
ffprobe -version
```

### A4. Python 3.12 nhưng plan ghi 3.11+

Máy có **Python 3.12.0** — thoả. Lưu ý: `datetime.utcnow()` bị deprecate ở 3.12, dùng `datetime.now(timezone.utc)`.

---

## B. MCP server

### B1. `initialize` fail ngay: `client_secret_file is required`

Binary bắt buộc flag `-client_secret_file` (`main.go`). Không có giá trị mặc định.

```text
youtube-uploader-mcp-windows-amd64.exe -client_secret_file "C:\...\client_secret.json" -working_dir "C:\...\secrets"
```

Kiểm tra: file tồn tại, là JSON có key `installed` hoặc `web`.

### B2. MCP thoát ngay, không log gì

- Chạy trực tiếp trong terminal để xem stderr. Nếu nó chờ stdin (stdio), treo là bình thường — dùng client MCP để test.
- `-version` in ra version rồi thoát: dùng để smoke test binary.

### B3. `initialize` báo version `0.1.3` dù tag là `v0.3.1`

Không phải lỗi. Release workflow build **không** truyền `-ldflags -X ...Version` ⇒ giá trị mặc định trong `yum/version.go` (`0.1.3`). Đừng dùng version này để phát hiện tính năng; dùng SHA-256 của binary hoặc kiểm tra `tools/list`.

### B4. `tools/list` thiếu `check_job_status`

Binary quá cũ (trước khi có tracker). Adapter phải **fail-fast**, không chạy tiếp. Tải đúng `v0.3.1`.

### B5. `job is not present in registry`

Tracker là **in-memory** (`tracker/tracker.go`). Nguyên nhân:

- MCP process vừa restart/crash.
- `job_key` gõ sai/không phải key do `upload_video` trả về.
- Job đã bị xoá khi process chết vì hết 10 phút.

Cách xử lý: **không** upload lại mù quáng. Chuyển `NEEDS_RECONCILIATION`, gọi `list_videos(channel_id, query=<title>)` để xem video đã tồn tại chưa.

### B6. Upload chạy mãi rồi lỗi `operation still in progress after 10m0s`

Hard timeout 10 phút (`upload_video.go:16-17`). Upload bị huỷ giữa đường ⇒ YouTube có thể đã tạo video **rỗng hoặc chưa hoàn chỉnh**. Bắt buộc reconciliation.

Giảm thiểu:

- Ước lượng `size / upload_speed`; nếu > 8 phút thì cảnh báo trước.
- Nén lại video (H.264 CRF hợp lý) hoặc chia nhỏ.
- Xem ADR-001 để quyết định direct API adapter.

### B7. Upload trả `{"status":"running","job_key":"..."}`

**Không phải lỗi.** Đó là early-return sau 30 giây. Phải poll `check_job_status(job_key)` cho tới khi có kết quả. Chi tiết ở `docs/architecture.md` §5.2.

### B8. Poll `check_job_status` trả text thay vì JSON

Khi job còn chạy, tool trả **plain text** `job is still running, please check in sometime with the same job_key`. Chỉ khi xong mới trả JSON kết quả gốc. Adapter phải xử lý cả hai.

### B9. `failed to get channels: failed to open channels file`

Chưa từng auth, nên `.youtube_uploader_channels_cache` chưa tồn tại, và `channels` gọi `ReadChannels(false)` nên trả lỗi. UI phải hiển thị "chưa kết nối kênh" thay vì coi là lỗi hệ thống.

### B10. `channel with ID ... not found`

`channel_id` không có trong cache. Có thể do:

- Xoá cache nhưng DB vẫn giữ kênh.
- Chọn sai `working_dir` ⇒ MCP đọc file khác.
- Kênh bị ngắt kết nối.

Kiểm tra `working_dir` của process có khớp `credential_ref` trong DB không.

---

## C. OAuth / quyền

### C1. `403 Forbidden` khi upload phụ đề

Thiếu scope `youtube.force-ssl` (token cũ trước khi scope này được thêm). Sửa:

1. Xoá `.youtube_uploader_channels_cache` (backup trước).
2. `authenticate` → mở URL → `Advanced` → `Go to … (unsafe)` → cấp quyền.
3. `accesstoken` với code mới.
4. Cập nhật `channels.status = CONNECTED`.

### C2. `403 insufficientPermissions` / `forbidden` khi `list_videos`

Token thiếu `youtube.readonly` hoặc không phải chủ kênh. Auth lại. Không retry vô hạn — đánh dấu `REAUTH_REQUIRED`.

### C3. `access_denied` ngay sau khi chọn tài khoản

Email chưa nằm trong **Test users** của OAuth consent screen. Thêm vào **Audience → Test users**.

### C4. Refresh token chết sau ~7 ngày

App đang ở trạng thái **Testing**. Google giới hạn refresh token 7 ngày. Publish consent screen (vẫn có thể unverified) hoặc chuyển sang Internal (Workspace).

Triệu chứng: upload lỗi refresh, `REAUTH_REQUIRED`, phải auth lại.

### C5. `redirect_uri_mismatch`

- Client loại **Web application** yêu cầu redirect URI khai báo chính xác.
- Client loại **Desktop app** cho phép loopback nhưng **port phải khớp** giữa URL auth và URL callback. Nếu backend chọn port động, phải dùng đúng port đó ở cả hai bước (`authenticate` và `accesstoken`).

### C6. Callback bị gọi 2 lần / replay

Code Google chỉ dùng một lần. Bảng `oauth_states` phải đánh dấu `used_at` **trước khi** gọi `accesstoken`; nếu không, request thứ hai sẽ lỗi exchange và khó hiểu.

---

## D. Dữ liệu / hành vi

### D1. Đăng trùng 2 lần cùng một video

Nguyên nhân đã biết: sau khi job thành công, gọi lại `upload_video` với cùng `(channel_id, file_path)` sẽ tạo **job mới**, vì tracker chỉ chặn khi job **đang chạy**.

Phòng: khoá theo `videos.state ∈ {UPLOADING, UPLOADED_PRIVATE, SCHEDULED, PUBLISHED}` và `youtube_video_id IS NOT NULL`. Không chỉ dựa vào `job_key`.

### D2. YouTube báo lỗi schedule dù giờ đúng

- Video phải `private` (MCP ép khi `publish_at` được set ở `upload_video`; với `update_video` thì từ chối nếu đang `public`).
- Timestamp phải **ở tương lai** tại thời điểm gọi.
- Kênh phải đủ điều kiện schedule của YouTube (tài khoản đã xác minh, không vi phạm). Đây là điều kiện phía YouTube, không sửa được bằng code.

### D3. Lệch múi giờ 7 tiếng

Dấu hiệu kinh điển: nhập 20:00, video publish lúc 13:00 hoặc 03:00. Nguyên nhân: gửi giờ local như thể UTC, hoặc quên offset. Luôn:

- Lưu UTC trong DB.
- Convert bằng `zoneinfo.ZoneInfo("Asia/Ho_Chi_Minh")`.
- Gửi RFC3339 có offset (`+07:00`) hoặc `Z`.
- Test riêng quanh mốc nửa đêm và khi VN không có DST (hiện không có, nhưng đừng hardcode +7 nếu muốn chạy nơi khác).

### D4. Video dài > 15 phút bị từ chối

YouTube yêu cầu kênh đã xác minh để upload video > 15 phút. Ngoài tầm kiểm soát của code; hiển thị hướng dẫn xác minh kênh.

### D5. Thumbnail bị từ chối

- File > 2MB: MCP chặn trước khi gọi API.
- Kênh chưa xác minh: YouTube từ chối thumbnail tuỳ chỉnh.
- Định dạng/kích thước: dùng JPEG/PNG, 1280×720, < 2MB.

### D6. Quota

`videos.insert` tốn rất nhiều unit; quota mặc định khiến số upload/ngày bị giới hạn. **Không hardcode con số 6** — hiển thị lỗi 403 kèm hướng dẫn xin tăng quota, và ghi lại thời điểm cạn quota để job `RETRY_WAIT` sang ngày hôm sau thay vì retry liên tục.

---

## E. Windows đặc thù

| Vấn đề | Xử lý |
|---|---|
| Đường dẫn có dấu cách/tiếng Việt | Luôn dùng list argv khi spawn subprocess, không nối chuỗi; kiểm tra encoding UTF-8 |
| Path > 260 ký tự | Bật `LongPathsEnabled` trong registry, hoặc giữ staging path ngắn |
| Windows Defender quét binary lạ | Có thể chậm lần đầu; thêm exclusion cho thư mục `bin/` nếu cần |
| Port bị chiếm | Kiểm tra `Get-NetTCPConnection -LocalPort 8000`; chọn port khác và cập nhật redirect |
| Process mồ côi khi backend crash | Worker phải kill MCP con khi tắt; dùng Job Object hoặc kiểm tra PID trước khi spawn |
| `chmod 0600` không có tác dụng | Trên Windows, mode chỉ là gợi ý. Phải set ACL bằng `icacls` cho thư mục secrets |

Kiểm tra ACL thư mục secrets:

```powershell
icacls "$env:LOCALAPPDATA\DangVideoTuDong\secrets"
# chỉ nên có: <user>:(OI)(CI)F  và SYSTEM:(OI)(CI)F
```
