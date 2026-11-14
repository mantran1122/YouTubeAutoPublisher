# YOUTUBE AUTO PUBLISHER — MASTER IMPLEMENTATION PLAN

> **Vai trò của tài liệu:** Đây là đặc tả và prompt triển khai cho Codex / Claude Code / AI coding agent. Hãy **đọc toàn bộ**, tạo mã nguồn thực tế, chạy thử, viết kiểm thử, sửa lỗi và bàn giao một ứng dụng hoạt động; **không** chỉ giải thích ý tưởng hoặc dừng ở scaffold.
>
> **Mục tiêu:** Xây dựng hệ thống đăng video YouTube tự động đa kênh, có web dashboard, nhận video từ máy tính hoặc Google Drive, tạo metadata/phụ đề bằng AI, đặt lịch, xếp hàng đợi và xử lý lỗi. **Ưu tiên Windows 10/11**; thiết kế để chạy được trên Linux.

## 0. Nguồn tham khảo bắt buộc

1. **Skill xử lý video**: https://github.com/smixs/youtube-publisher
   - Đọc `README.md`, `SKILL.md`, `scripts/` và `LICENSE`.
   - Tham khảo luồng tải từ Google Drive, tách âm thanh với ffmpeg, transcription (Fireworks/Deepgram), tạo chapters, title và description.
   - Theo README, CLI hiện tập trung vào URL Google Drive; **không giả định** nó upload trực tiếp file local.
2. **MCP đăng YouTube**: https://github.com/anwerj/youtube-uploader-mcp
   - Đọc `README.md`, `youtube_oauth2_setup.md`, mã Go và `LICENSE`.
   - Tìm hiểu các MCP tools thực tế: `authenticate`, `accesstoken`, `channels`, `refreshtoken`, `upload_video`, `update_video`, `list_videos`.
   - Tham khảo OAuth2 đa kênh, upload, cập nhật thumbnail/subtitle/playlist, privacy, schedule.

**Quy tắc tích hợp:** Hai repository có cấu trúc khác nhau (Python agent skill và Go MCP server), không phải hai thư viện Python có thể `import` giống nhau. Clone vào `external/`, khóa commit/version, xem kỹ giao diện thực tế và chọn adapter phù hợp. Không sao chép nguyên khối code khi không cần. Ghi nhận LICENSE/attribution. Không chạy script cài đặt tải từ Internet mà không xem nội dung.

## 1. Kết quả cuối cùng phải bàn giao

Một dự án chạy được từ máy Windows, có:
- Dashboard web tiếng Việt, responsive, cho phép thao tác mà **không cần gọi AI trong mỗi lần đăng**.
- Kết nối và quản lý nhiều kênh YouTube qua OAuth2.
- Nhập video local (chọn file hoặc drag-and-drop) và nhập URL Google Drive khi có quyền truy cập hợp lệ.
- Danh sách video nháp, thông tin video, xem trước, thumbnail, transcript, chapters, phụ đề.
- Tự tạo/gợi ý tiêu đề, mô tả, hashtag/tags, chapters theo nội dung bằng nhà cung cấp AI có cấu hình; luôn cho phép sửa bằng tay.
- Hàng đợi upload tự động, đặt lịch theo múi giờ `Asia/Ho_Chi_Minh`, retry an toàn, báo tiến độ và trạng thái.
- Danh sách lịch đăng dạng bảng và lịch; lịch sử upload, lỗi, nhật ký và link video YouTube.
- Hỗ trợ một kênh trước, nhiều kênh qua từng kết nối OAuth độc lập; tránh đăng nhầm kênh.
- Chạy được khi trình duyệt đóng; không phụ thuộc vào phiên Claude/Codex đang mở.
- README hướng dẫn cài đặt, `.env.example`, script khởi chạy Windows, bộ test cơ bản, tài liệu vận hành.

**Không yêu cầu ở bản đầu:** tự sản xuất video, tự đăng bình luận hàng loạt, tương tác giả, tải lại video không có quyền, vượt giới hạn API.

## 2. Công nghệ đề xuất

| Lớp | Chọn mặc định | Ghi chú |
|---|---|---|
| Frontend | React + TypeScript + Vite + Tailwind CSS | Dashboard rõ ràng, dễ mở rộng |
| Backend | Python 3.11+ + FastAPI + Pydantic | Điều phối và API |
| Database | SQLite + SQLAlchemy + Alembic | Chạy local; dễ chuyển PostgreSQL |
| Worker/scheduler | Worker nền riêng + APScheduler hoặc cơ chế DB-backed đáng tin cậy | Job được lưu DB, phục hồi sau restart |
| Upload engine | Adapter cho `youtube-uploader-mcp` | Kết nối theo giao thức MCP thực tế qua subprocess/stdio; không tự giả định HTTP API |
| Media/AI engine | Adapter Python tham khảo `youtube-publisher`, ffmpeg/ffprobe | Tách âm thanh, Drive, transcript, chapters |
| AI metadata | Interface provider: OpenAI-compatible API hoặc provider khác | API key tùy chọn, có fallback thủ công |
| Logging | Python logging JSON/structured | Không lộ token/secret |
| Tests | pytest + kiểm thử API + Playwright smoke test | Mock Google API trong CI |

Có thể thay đổi stack nếu có lý do kỹ thuật rõ ràng, nhưng phải ghi ADR trong `docs/decisions/` và không đánh đổi khả năng chạy local Windows.

## 3. Kiến trúc mục tiêu

```text
Browser (React dashboard)
            |
            v
       FastAPI REST/SSE
            |
    +-------+---------+----------------+
    |                 |                |
SQLite/DB       Job scheduler      Secret store
    |                 |                |
    +---------- Queue/Worker ----------+
                      |
           +----------+-----------+
           |                      |
     Media adapter           YouTube adapter
 (Drive, ffmpeg, ASR, AI)    (MCP Go server)
           |                      |
      Local storage            YouTube API
```

- Giao diện chỉ giao tiếp với backend; OAuth token, API key không gửi xuống frontend.
- Dùng `YoutubeProvider`/`UploadAdapter` abstraction: triển khai MCP client thật (tool discovery, schema validation, lifecycle process, timeout). Chỉ bổ sung direct YouTube API adapter nếu MCP không đáp ứng resumable progress/refresh/reliability; ghi rõ chức năng nào dùng adapter nào.
- Tránh khởi động một phiên MCP server cho từng request nếu không cần; quản lý vòng đời subprocess và xử lý crash/reconnect.
- **AI không trực tiếp giữ token YouTube**, chỉ nhận transcript/metadata cần thiết; hạn chế PII và ghi log.

## 4. Luồng nghiệp vụ

### A. Thêm video
1. Người dùng chọn file mp4/mov/mkv/webm hoặc nhập Drive link có quyền truy cập.
2. Backend kiểm tra file size, extension/MIME thực tế, định dạng bằng ffprobe, thời lượng và dung lượng ổ đĩa.
3. Với Drive: dùng OAuth `drive.readonly`, tải dạng streaming vào vùng staging; báo tiến độ nếu SDK hỗ trợ.
4. Tính SHA-256 hoặc fingerprint phù hợp; ghi DB để ngăn import trùng trong cùng phạm vi kênh theo cấu hình.
5. Tạo bản ghi `DRAFT` và thumbnail preview local, không public URL chứa secret.

### B. Phân tích và gợi ý nội dung
1. Tách audio qua ffmpeg; chia thành chunks phù hợp nếu cần.
2. Chạy ASR với Fireworks/Deepgram theo cấu hình, hoặc provider thay thế; **nếu chưa cấu hình API**, vẫn cho nhập title/description thủ công và upload.
3. Tạo transcript theo thời gian, gợi ý chapters hợp lệ; kiểm tra chapters không vượt quá duration.
4. Dùng AI sinh **3 phương án** tiêu đề, mô tả, tags, hashtags; hỗ trợ ngôn ngữ tiếng Việt/Anh.
5. Không tự bịa nội dung, không nhồi từ khóa, không tự khẳng định bản quyền; yêu cầu người dùng xem trước hoặc bật chế độ tự duyệt có chủ ý.
6. Lưu transcript/subtitle `.srt` hoặc `.vtt`; cho phép người dùng sửa và gắn phụ đề.

### C. Lên lịch và upload
1. Người dùng chọn kênh, visibility (`private` / `unlisted` / `public`) và hình thức đăng `now` / `scheduled`.
2. Với `scheduled`, nhập giờ theo `Asia/Ho_Chi_Minh`; chuyển UTC/RFC3339 và xác thực thời điểm trong tương lai.
3. Khi tới thời điểm **bắt đầu upload**, job được worker lấy từ DB. Phân biệt rõ **upload time** và **YouTube publishAt**; nên upload video ở chế độ private trước giờ công bố để có thời gian xử lý, sau đó cập nhật `publishAt` theo điều kiện API.
4. Kiểm tra kênh và scope OAuth, file còn tồn tại, title/description, quota và trạng thái job trước khi upload.
5. Dùng `upload_video` của MCP adapter (hoặc direct API nếu có ADR giải thích), nhận video ID và lưu ngay khi có.
6. Dùng `update_video` gắn thumbnail, phụ đề, playlist, flags về trẻ em và nội dung biến đổi/tổng hợp (chỉ theo lựa chọn người dùng), lịch publish nếu cần.
7. Đối chiếu video ID/trạng thái sau upload. Nếu lỗi sau khi YouTube đã tạo video, **không được upload lại mù quáng**; đưa vào `NEEDS_RECONCILIATION` và tra cứu/đối chiếu trước.
8. Trạng thái `UPLOADED_PRIVATE`, `SCHEDULED`, `PUBLISHED`, `FAILED` phản ánh thực tế, không đánh dấu `PUBLISHED` chỉ vì API upload trả thành công.

### D. Xử lý lỗi
- Retry có exponential backoff + jitter cho lỗi tạm thời (network, 429, 5xx), giới hạn số lần; tôn trọng Retry-After nếu có.
- 401/token expired: thử refresh chính thống; refresh thất bại -> `REAUTH_REQUIRED`.
- 403/quota/scopes/permission: hiển thị lỗi có thể hành động, không retry vô hạn.
- Crash/mất điện: worker khởi động lại, reclaim job quá hạn lease, đối chiếu video ID trước khi tiếp tục.
- Hủy job chưa upload hoặc đang queue; xác định giới hạn của việc hủy upload đang chạy.
- Mỗi job có `idempotency_key`, lock/lease và audit log tránh nhiều worker đăng cùng một video.

## 5. Module và màn hình bắt buộc

### Frontend
1. **Tổng quan**: số nháp, chờ đăng, đang tải, đã đăng, lỗi; công việc gần đây.
2. **Kênh YouTube**: kết nối OAuth, hiển thị channel ID/tên/ảnh (nếu API trả), trạng thái token, ngắt kết nối; xác nhận kênh trước upload.
3. **Thư viện video**: import local/Drive, tìm kiếm, filter, preview, chỉnh sửa metadata.
4. **AI Studio**: transcript, chapters, chọn title/mô tả/tags gợi ý, chỉnh thủ công, lựa chọn ngôn ngữ.
5. **Hàng đợi / lịch đăng**: ngày giờ, kênh, visibility, kéo đổi thứ tự ưu tiên an toàn, tạm dừng/tiếp tục.
6. **Lịch sử**: trạng thái, video ID, link YouTube, thời gian, lỗi, nút retry có kiểm tra.
7. **Cài đặt**: API keys (masked), Drive, định dạng mặc định, timezone, giới hạn song song, folder staging.
8. **Logs**: xem dòng sự kiện của một job, copy thông báo lỗi đã lọc secret.

UX: giao diện tiếng Việt mặc định; loading/skeleton, empty states, xác nhận hành động nguy hiểm, toast, validation field, dark mode tùy chọn. Không dùng giao diện demo giả và nút không hoạt động.

### Backend API (đề xuất)

```text
GET    /api/health
GET    /api/channels
POST   /api/channels/oauth/start
GET    /api/channels/oauth/callback
DELETE /api/channels/{id}
GET    /api/videos
POST   /api/videos/import-local
POST   /api/videos/import-drive
GET    /api/videos/{id}
PATCH  /api/videos/{id}
POST   /api/videos/{id}/analyze
POST   /api/videos/{id}/generate-metadata
POST   /api/videos/{id}/schedule
POST   /api/videos/{id}/upload-now
GET    /api/jobs
GET    /api/jobs/{id}
POST   /api/jobs/{id}/retry
POST   /api/jobs/{id}/cancel
GET    /api/jobs/{id}/events
GET    /api/settings
PATCH  /api/settings
```

Dùng SSE hoặc WebSocket cho tiến độ. Định nghĩa OpenAPI schemas và trả mã HTTP nhất quán.

## 6. Thiết kế dữ liệu tối thiểu

- `channels`: id, youtube_channel_id, display_name, status, credential_ref, created_at.
- `videos`: id, source_type, source_uri_or_path, fingerprint, file_size, duration, title, description, tags_json, category_id, language, audience_flag, synthetic_content_flag, privacy, thumbnail_path, transcript_path, subtitle_path, youtube_video_id, created_at, updated_at.
- `jobs`: id, video_id, channel_id, kind, status, scheduled_at_utc, upload_start_at_utc, next_run_at_utc, attempts, max_attempts, idempotency_key, lease_owner, lease_until, last_error_code, last_error_detail, created_at, updated_at.
- `job_events`: id, job_id, level, event_type, message_redacted, timestamp.
- `settings`: key, value_encrypted_or_plain_by_type, updated_at.
- `ai_generations`: id, video_id, provider, prompt_version, output_json, created_at.

Tạo migration versioned. Cấm lưu access/refresh token plaintext trong DB/log. Nếu triển khai local, dùng Windows DPAPI/OS credential store hoặc encryption key ngoài DB, nêu rõ phạm vi bảo vệ và hướng dẫn backup.

## 7. Bảo mật và tính đúng đắn

- OAuth 2.0 Authorization Code/PKCE khi phù hợp với loại client Google; scope tối thiểu. Google Cloud Console cần bật YouTube Data API v3 và Drive API nếu dùng Drive.
- Không hardcode `client_secret`, `refresh_token`, ASR/LLM API key. `.env` và credential files phải được gitignore.
- Kiểm tra permission của tài khoản/kênh sau OAuth. Không coi email Google tương đương channel ID.
- API web có xác thực local admin hoặc session security nếu expose ngoài localhost. Mặc định bind `127.0.0.1` và cấu hình CORS chặt chẽ.
- Chống path traversal, file zip-bomb tương đương, symlink escape, upload file lạ, SSRF qua URL ngoài whitelist (Drive chỉ chấp nhận link Google hợp lệ).
- Không gửi credentials hoặc toàn bộ đường dẫn nhạy cảm vào prompt AI.
- Giới hạn dung lượng, số upload song song và chi phí phiên âm/AI; cấu hình retention và cleanup cho file tạm.
- Tôn trọng YouTube API Services Terms, quota, quy định quyền sở hữu nội dung; không thiết kế né hạn chế hoặc spam đa kênh.
- Thông tin quota/chính sách có thể thay đổi: **kiểm tra tài liệu chính thức khi triển khai**, không hardcode giới hạn 6 video/ngày chỉ vì README cũ.
- Video `scheduled` cần bảo đảm điều kiện API cho `publishAt` (private và yêu cầu liên quan); trình bày rõ nếu YouTube chưa cho phép schedule.

## 8. Cấu trúc thư mục đề xuất

```text
youtube-auto-publisher/
├── README.md
├── .env.example
├── .gitignore
├── docker-compose.yml             # optional
├── scripts/
│   ├── setup_windows.ps1
│   ├── start_windows.bat
│   └── start_linux.sh
├── backend/
│   ├── pyproject.toml
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── auth/
│   │   ├── config/
│   │   ├── db/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── services/
│   │   │   ├── importers/
│   │   │   ├── media/
│   │   │   ├── ai/
│   │   │   └── youtube/
│   │   ├── workers/
│   │   └── security/
│   ├── migrations/
│   └── tests/
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── features/
│   │   └── api/
│   └── tests/
├── external/
│   ├── youtube-publisher/
│   └── youtube-uploader-mcp/
├── storage/                       # gitignored
│   ├── staging/
│   ├── thumbnails/
│   ├── transcripts/
│   └── logs/
└── docs/
    ├── architecture.md
    ├── oauth-setup.md
    ├── troubleshooting.md
    ├── test-report.md
    └── decisions/
```

## 9. Trình tự triển khai bắt buộc cho AI coding agent

### Giai đoạn 0 — Audit upstream
- [ ] Clone hai repository và ghi SHA commit đang dùng.
- [ ] Đọc `SKILL.md`, source Python, source Go, các hướng dẫn OAuth, licenses.
- [ ] Xác định tool schemas MCP thực tế, giao thức MCP, khả năng quản lý process, giới hạn file local, thumbnail, captions, scheduling.
- [ ] Viết `docs/architecture.md` và bảng **Supported / Needs adapter / Not supported**.
- [ ] Nêu rủi ro thư viện, cách quản lý upstream updates; không tuyên bố đã test OAuth nếu chưa có tài khoản test.

### Giai đoạn 1 — Minimum usable product (ưu tiên chạy được)
- [ ] Tạo backend/frontend, database + migrations, scripts khởi chạy Windows.
- [ ] Kết nối OAuth YouTube một kênh bằng flow an toàn và hiển thị thông tin kênh.
- [ ] Import file local, điền metadata, thumbnail tùy chọn.
- [ ] Upload thủ công một video test ở chế độ **private**; lưu video ID.
- [ ] Dashboard hiển thị trạng thái và lỗi thực tế; restart ứng dụng không làm mất records.
- [ ] Viết hướng dẫn cấu hình và kiểm thử với mock trước khi gọi API thực.

### Giai đoạn 2 — Automation core
- [ ] Worker nền với DB-backed queue và trạng thái jobs.
- [ ] Lịch upload, lịch công bố trên YouTube, phân biệt 2 mốc thời gian.
- [ ] Retry/backoff, recovery sau restart, dedupe, reauthentication.
- [ ] `update_video`: thumbnail, captions, playlist, privacy, audience/synthetic flags khi API hỗ trợ.
- [ ] Multiple channels với token tách biệt; bắt buộc kiểm tra channel ID ở mọi job.

### Giai đoạn 3 — AI pipeline và Google Drive
- [ ] Adapter Drive download/authorization.
- [ ] ffmpeg + transcript provider từ mẫu youtube-publisher.
- [ ] AI title/description/tags, chapters, subtitle.
- [ ] Hàng đợi xử lý media và upload riêng; cache kết quả; có kiểm soát chi phí.
- [ ] Cho phép thao tác hoàn toàn thủ công khi AI API không hoạt động.

### Giai đoạn 4 — Hoàn thiện và kiểm thử
- [ ] E2E test UI: import → edit → schedule → queue → complete (mock provider).
- [ ] Integration tests cho MCP tool client với mock responses.
- [ ] Test timezone, token expired, crash, network disconnect, duplicate attempts, invalid Drive link, wrong channel, quota exhaustion.
- [ ] Kiểm tra không lộ secret, CORS/CSRF phù hợp, upload file validation, schema migration.
- [ ] README và hướng dẫn Windows từ máy sạch; verify commands, logs, test report.
- [ ] Nếu có credentials test được cấp rõ ràng, thực hiện một lượt upload PRIVATE và báo cáo URL/video ID; không tự public video để thử.

## 10. Checklist nghiệm thu

- [ ] `setup_windows.ps1` chuẩn bị môi trường; `start_windows.bat` chạy backend/frontend/worker theo cách được ghi tài liệu.
- [ ] Mở dashboard qua trình duyệt, không có dead button hoặc data giả.
- [ ] OAuth liên kết đúng kênh; giữ đăng nhập sau restart; secret không lộ.
- [ ] Upload file local thành công với metadata đầy đủ, trả video ID.
- [ ] Import từ Drive hoạt động sau khi kết nối Drive hợp lệ.
- [ ] Lịch theo giờ Việt Nam chính xác, không nhầm UTC; private schedule làm đúng quy tắc API.
- [ ] Có thể chuẩn bị nhiều video trong queue; worker không upload trùng sau lỗi.
- [ ] Retry và báo lỗi có thể hiểu; mạng mất/restored xử lý đúng.
- [ ] AI transcription và gợi ý nội dung có thể bật/tắt; fallback nhập tay.
- [ ] Thumbnail, playlist và phụ đề hoạt động nếu được cấp scopes đầy đủ.
- [ ] Database, logs, file staging có chính sách backup/cleanup.
- [ ] Tests pass; không có lỗi lint/typecheck/build nghiêm trọng.
- [ ] README cho biết cụ thể tính năng **đã chạy thật**, **đã mock**, **cần OAuth**, **chưa hỗ trợ**.

## 11. Các lệnh và hướng dẫn phải tạo ra trong README

Cung cấp lệnh cụ thể để:
1. Clone project cùng các dependency upstream (hoặc checkout locked commit).
2. Cài Python/Node/Go hoặc prebuilt binary MCP an toàn trên Windows.
3. Cài ffmpeg và cấu hình PATH.
4. Thiết lập `.env`, Google OAuth client, scopes và redirect URI chính xác.
5. Start/stop backend, frontend, worker; giải thích cách hoạt động khi đóng browser.
6. Chạy unit/integration/e2e tests.
7. Backup DB, xóa video tạm an toàn, refresh/reconnect OAuth và đọc logs.

Ví dụ `.env.example` **chỉ là biến mô tả**; điều chỉnh theo code thực:

```dotenv
APP_ENV=development
APP_HOST=127.0.0.1
APP_PORT=8000
APP_TIMEZONE=Asia/Ho_Chi_Minh
DATABASE_URL=sqlite:///./storage/app.db
STORAGE_DIR=./storage
YOUTUBE_OAUTH_CLIENT_FILE=./secrets/google-oauth-client.json
YOUTUBE_MCP_EXECUTABLE=./bin/youtube-uploader-mcp-windows-amd64.exe
GOOGLE_DRIVE_ENABLED=false
FIREWORKS_API_KEY=
DEEPGRAM_API_KEY=
AI_PROVIDER=
AI_API_KEY=
MAX_CONCURRENT_UPLOADS=1
```

## 12. Quy tắc làm việc với AI coding agent

- **Bắt đầu lập trình ngay sau audit**, không chỉ viết kế hoạch.
- Chủ động quyết định chi tiết phụ trong phạm vi tài liệu; không hỏi lại những thông tin đã có.
- Khi thiếu OAuth/API key thật: hoàn thành UI, backend, adapters và automated tests bằng mock; hướng dẫn người dùng điền credentials để test thực. **Không giả lập là đã đăng video thật.**
- Làm theo từng phase có thể chạy được, nhưng mục tiêu cuối là hệ thống hoàn chỉnh. Sau mỗi phase: chạy build/test, sửa lỗi rồi mới tiếp tục.
- Ghi lại mọi khác biệt với README upstream (tool/schema/flag) sau khi xem source code; **không bịa API**.
- Tối ưu UX, vận hành ổn định và độ an toàn hơn số lượng tính năng trình diễn.
- Khi kết thúc: báo cáo file đã tạo, cách chạy, kết quả test, hạn chế và những bước cấu hình còn cần người dùng thực hiện.

## 13. Prompt kích hoạt cho Codex / Claude Code

> Hãy đọc `YOUTUBE_AUTO_PUBLISHER_PLAN.md` và triển khai **toàn bộ hệ thống YouTube Auto Publisher** theo tài liệu. Trước tiên clone/audit hai repo tham chiếu (youtube-publisher và youtube-uploader-mcp), xác minh tool interfaces từ source, rồi triển khai theo các giai đoạn 0–4. Tôi sử dụng Windows và cần web dashboard tiếng Việt. Chủ động code, chạy build/test, sửa lỗi, cập nhật README; khi thiếu Google OAuth/API keys thì dùng mock test và nêu chính xác cách cấu hình, tuyệt đối không khai đã upload thật. Không dừng ở bản mô tả hoặc scaffold. Ưu tiên MVP chạy được trước, sau đó hoàn thiện queue, scheduling, multi-channel, AI processing và Google Drive.

---

**Nguồn chính:**
- https://github.com/smixs/youtube-publisher
- https://github.com/anwerj/youtube-uploader-mcp
- Tài liệu YouTube Data API chính thức: https://developers.google.com/youtube/v3

**Lưu ý:** Thông tin về tính năng và quota có thể thay đổi theo phiên bản/revision; luôn đọc mã nguồn và tài liệu chính thức tại thời điểm thực thi.
