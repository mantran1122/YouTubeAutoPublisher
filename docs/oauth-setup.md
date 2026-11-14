# OAuth setup — Google Cloud, YouTube, nhiều kênh

> Mục tiêu: tạo `client_secret.json`, cấp đúng scope, luồng callback an toàn, và cách xử lý nhiều kênh.
> Chưa test với tài khoản thật ở thời điểm viết.

---

## 1. Tạo Google Cloud project

1. Vào <https://console.cloud.google.com> → tạo project mới (ví dụ `YouTubeAutoPublisher`).
2. Bật **YouTube Data API v3**: <https://console.cloud.google.com/apis/library/youtube.googleapis.com>
3. Nếu dùng Google Drive: bật **Google Drive API**: <https://console.cloud.google.com/apis/library/drive.googleapis.com>
4. Vào **APIs & Services → OAuth consent screen**:
   - User type: **External**.
   - Điền app name, email.
   - **Không** cần khai báo scope ở bước này — scope xin lúc chạy.
5. Vào **Audience → Test users**, thêm **đúng email Google** của kênh YouTube sẽ dùng. Thiếu bước này ⇒ Google chặn với lỗi `access_denied`.

## 2. Tạo OAuth client

1. **Credentials → Create credentials → OAuth client ID**.
2. Application type: **Desktop app**.
3. Tải JSON về, đặt tên `client_secret.json`.
4. Lưu **ngoài repo**, ví dụ:

```text
C:\Users\<user>\AppData\Local\DangVideoTuDong\secrets\client_secret.json
```

Không commit. Không để trong `storage/`. Không gửi qua chat/email.

Với **Desktop app**, Google cho phép redirect URI dạng loopback (`http://127.0.0.1:<port>`) **không cần khai báo trước**. Điều này cho phép backend của ta dùng callback riêng:

```text
http://127.0.0.1:8000/api/channels/oauth/callback
```

Nếu dùng loại **Web application** thay vì Desktop, phải khai báo redirect URI chính xác trong Console, nếu không sẽ `redirect_uri_mismatch`. Khuyến nghị dùng Desktop app cho app local.

## 3. Scope

MCP server xin cố định 3 scope (`core/core.go:41-46`), ta **không** thay đổi được qua tham số:

| Scope | Dùng cho |
|---|---|
| `https://www.googleapis.com/auth/youtube.upload` | Upload video |
| `https://www.googleapis.com/auth/youtube.force-ssl` | Caption/subtitle, playlist, `videos.update` (status), thumbnail |
| `https://www.googleapis.com/auth/youtube.readonly` | `channels.list`, `playlistItems.list` để liệt kê/đối chiếu |

Hệ quả:

- `youtube.force-ssl` là scope **nhạy cảm** ⇒ Google hiện cảnh báo "Google hasn't verified this app". Với app cá nhân: **Advanced → Go to … (unsafe)**.
- Token cũ (trước khi có `force-ssl`) sẽ **403** khi upload phụ đề. Cách sửa: xoá `.youtube_uploader_channels_cache` rồi auth lại.
- Không có scope `youtube` đầy đủ ⇒ **không** xoá video, không sửa comment, không đọc analytics.

Nếu dùng Drive, `youtube-publisher` dùng bộ scope **khác** (`drive.readonly`, `youtube.upload`, `youtube`). Đó là client/token riêng — **không dùng chung** với MCP. Cân nhắc tạo một OAuth client riêng cho Drive, hoặc một client với union scope và hai token file tách biệt.

## 4. Luồng OAuth của ta (bù lỗ hổng state của MCP)

Vấn đề: `core/oauth.go:25` dùng `state` cố định `"state-token"` ⇒ không chống CSRF, không chống replay.

Thiết kế bắt buộc:

```text
1. POST /api/channels/oauth/start
   → backend sinh nonce ngẫu nhiên (32 bytes), lưu server-side:
       nonce, created_at, expires_at = now + 10 phút, used = false
   → gọi MCP authenticate(redirect_uri = "http://127.0.0.1:8000/api/channels/oauth/callback")
   → nhận auth_url, trả về cho trình duyệt
   → KHÔNG trả nonce xuống client dưới dạng query param của ta

2. Người dùng đăng nhập Google, cấp quyền

3. GET /api/channels/oauth/callback?code=...&state=state-token
   → kiểm tra: có đúng 1 nonce đang mở (hoặc khớp cookie session), chưa hết hạn, chưa dùng
   → đánh dấu used = true NGAY (one-time)
   → gọi MCP accesstoken(code, redirect_uri = cùng URI)
   → nhận channel_id, tên kênh (token đã mask, không bao giờ về browser)
   → ghi DB: channels.id, youtube_channel_id, display_name, credential_ref
   → redirect về dashboard, không kèm code

4. Nếu state/code không hợp lệ → 400, không gọi accesstoken
```

Lưu ý:

- `state` từ Google sẽ **luôn** là `"state-token"`; đừng dùng nó để xác thực. Xác thực bằng nonce/session của ta.
- Code Google dùng một lần và hết hạn rất nhanh — đừng retry `accesstoken` với cùng code.
- Nếu người dùng mở nhiều tab, mỗi lần `start` phải tạo nonce riêng; callback khớp nonce nào thì dùng nonce đó.
- Không log `code`, không log URL đầy đủ.

## 5. Nhiều kênh

- Mỗi lần auth xong, MCP lưu **thêm** một entry vào `.youtube_uploader_channels_cache` (key = channel ID). Không ghi đè kênh cũ.
- DB của ta lưu `channels.youtube_channel_id` = ID thật từ Google, **không** dùng email.
- Mọi job **bắt buộc** lưu `channel_id` ở DB; trước khi upload phải:
  1. `channels` → xác nhận channel ID còn tồn tại và token còn hạn.
  2. So `job.channel_id` với `video.target_channel_id`, lệch ⇒ fail, không upload.
- Không cho phép "kênh mặc định" ngầm định khi có > 1 kênh.
- Ngắt kết nối: xoá entry khỏi DB của ta; **và** phải gỡ kênh khỏi `.youtube_uploader_channels_cache` (MCP không có tool xoá kênh) — thao tác file có kiểm soát, backup trước.

## 6. Refresh token

- MCP tự refresh khi token còn < 2 phút, trong `upload_video` / `update_video` / `list_videos`, và **lưu lại** file.
- `refreshtoken` tool **không đáng tin**: trả `Invalid access token loaded` khi `ExpiresIn == 0` (`yum/tool/refresh_token.go:49-51`), mà token đọc từ file luôn có `ExpiresIn == 0`. ⇒ Không dùng tool này; để upload tự refresh.
- Refresh fail (revoked, password changed, 7-day testing-token expiry) ⇒ `REAUTH_REQUIRED`, hiển thị nút "Kết nối lại kênh".
- **Lưu ý quan trọng**: app ở chế độ **Testing** trên Google Cloud có refresh token hết hạn sau **7 ngày**. Nếu muốn chạy dài hạn, phải **Publish app** (vẫn có thể ở trạng thái unverified với cảnh báo) hoặc dùng Internal nếu có Workspace.

## 7. Chuyển sang production (khi cần)

| Vấn đề | Cách xử lý |
|---|---|
| Cảnh báo "unverified app" | Chấp nhận với app cá nhân; hoặc submit verification nếu phân phối cho người khác |
| Refresh token hết hạn 7 ngày | Publish OAuth consent screen (không cần verify nếu <100 user và chỉ dùng nội bộ) |
| Sensitive scope review | `youtube.force-ssl` + `youtube.upload` có thể bị yêu cầu giải trình nếu mở công khai |
| Quota API | Mặc định 10.000 unit/ngày; `videos.insert` tốn ~1600 unit ⇒ ~6 upload/ngày. Kiểm tra lại tại thời điểm triển khai, **không hardcode** vào code |

## 8. Checklist trước khi bấm "Kết nối kênh"

- [ ] Đã bật YouTube Data API v3.
- [ ] OAuth client là **Desktop app**, file JSON hợp lệ (`installed` hoặc `web`).
- [ ] Email kênh nằm trong **Test users**.
- [ ] `client_secret.json` nằm ngoài repo, ACL chỉ user hiện tại.
- [ ] Backend bind `127.0.0.1`, callback đúng port.
- [ ] Nonce one-time + hết hạn 10 phút đã implement.
- [ ] Sau auth: DB có channel ID thật, response không chứa token.
- [ ] Kiểm tra `.youtube_uploader_channels_cache` có mode/ACL hạn chế.
