# Service map: Short Video

id: service-map:short-video
module: module:short-video
deployable: core-api
confidence: medium

Video ngắn: upload, duyệt, xem, tim, bình luận

Sở hữu vòng đời Video (uploading -> processing -> pending_review|published -> removed|rejected|failed), view/reaction/comment và ranking v1. Upload đi qua VideoStoragePort/VideoTranscodePort RIÊNG của module (KHÔNG dùng module Media); không sở hữu graph bạn bè, report/trust (Safety), hồ sơ tác giả (User) hay tiền quà tặng (Gift/Economy).

Owns: bảng videos (status, storageKey, rankScore, viewCount/likeCount/commentCount); bảng video_views (unique videoId+viewerId, qualified), video_reactions, video_comments; cờ VIDEO_UPLOAD_ENABLED / VIDEO_MODERATION_MODE (pre|post)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/videos/upload-intent | jwt | Tạo Video{uploading} + presigned uploadUrl qua VideoStoragePort; bắt buộc Idempotency-Key (replay phát lại ĐÚNG storageKey, khác caption 409); bị chặn 403 khi VIDEO_UPLOAD_ENABLED=false. |
| e2 | rest | POST | /api/v1/videos/:id/finalize | jwt | Client báo upload xong: conditional UPDATE uploading -> processing, gọi VideoTranscodePort.transcode rồi sang pending_review\|published theo VIDEO_MODERATION_MODE; lỗi transcode -> failed. Chỉ tác giả. |
| e3 | rest | GET | /api/v1/videos | jwt | Danh sách video published (for_you hoặc following = video của bạn bè qua listFriendIds; sort recent\|ranked) kèm author PublicProfileDto batch; bạn chưa có ai -> trang rỗng. |
| e4 | rest | GET | /api/v1/videos/:id | jwt | Chi tiết 1 video qua getVideoOrThrow: chỉ tác giả hoặc video published, còn lại 404 VIDEO_NOT_FOUND. |
| e5 | rest | POST | /api/v1/videos/:id/report | jwt | Report video: kiểm video visible rồi SafetyService.reportVideo (unique 1 report/người/video); vượt VIDEO_REPORT_AUTOHIDE_THRESHOLD distinct reporter thì tự published -> removed. |
| e6 | rest | POST | /api/v1/videos/:id/views | jwt | Ghi watch-time: self-view bỏ qua, video_views unique, viewCount chỉ +1 đúng lúc qualified chuyển false->true (cùng transaction). |
| e7 | rest | POST | /api/v1/videos/:id/reactions | jwt | Thả tim idempotent: insert VideoReaction + tăng likeCount cùng transaction, thua race unique thì replay không đếm đôi. |
| e8 | rest | DELETE | /api/v1/videos/:id/reactions | jwt | Bỏ tim idempotent: xoá VideoReaction và chỉ giảm likeCount khi xoá được dòng. |
| e9 | rest | POST | /api/v1/videos/:id/comments | jwt | Bình luận video: insert VideoComment + tăng commentCount cùng transaction, kèm author PublicProfileDto. |
| e10 | rest | GET | /api/v1/videos/:id/comments | jwt | Liệt kê comment theo cursor sau khi qua getVideoOrThrow, batch author PublicProfileDto. |
| e11 | rest | DELETE | /api/v1/videos/:id/comments/:commentId | jwt | Xoá mềm bình luận (set deletedAt): chỉ tác giả comment hoặc tác giả video được xoá, người khác 403, comment đã xoá 404; commentCount giảm 1 vô điều kiện trong cùng transaction. |
| e12 | job | WORKER | VideoRankingService.onApplicationBootstrap | internal | Job VideoRankingService tính lại rankScore (view/like/comment có trọng số, giảm theo tuổi) cho video published theo lô; derive, chạy 1 instance toàn cluster, lỗi job chỉ làm ranking cũ đi. |
| e13 | job | WORKER | VideoSweeperService.onApplicationBootstrap | internal | Job VideoSweeperService đánh failed các video kẹt uploading quá VIDEO_UPLOAD_TIMEOUT_SECONDS rồi gọi VideoStoragePort.delete dọn object mồ côi của video failed (retry ở tick sau nếu lỗi). |

## Coordination edges (why this service talks to others)

### graph bạn cho tab following (c1) with friend [service, out, di-call]

- endpoints: e3
- what: FriendService.listFriendIds(userId) cho GET /videos?feed=following để giới hạn authorUserId IN (bạn bè); rỗng thì trả trang rỗng, không query videos.
- why: Hệ thống không có bảng follow riêng: graph bạn bè do Friend sở hữu là nguồn 'theo dõi' duy nhất (docs/services/short-video-service.md § 3b). Short-video không thể tự suy ra ai là bạn mà không sao chép dữ liệu Friend.
- if missing: Tab 'Đang theo dõi' không có nguồn dữ liệu: phải trả toàn bộ video hoặc tự giữ bản sao bạn bè dễ lệch với Friendship thật.
- evidence: apps/core-api/src/modules/short-video/short-video.service.ts:206
- review: sound — listFriendIds trả rỗng thì trả trang rỗng, không query videos. Giới hạn 500 bạn nên bạn quá số đó bị bỏ.

### ghi report video + ngưỡng (c2) with safety [service, out, di-call]

- endpoints: e5
- what: SafetyService.reportVideo(reporterId, videoId, reason, description) trả distinctReporterCount; short-video so với VIDEO_REPORT_AUTOHIDE_THRESHOLD rồi autoHideIfPublished (published -> removed).
- why: Report/Trust là dữ liệu của Safety (mở rộng Report targetType='video', unique reporter+video, KHÔNG trừ trust score). Safety trung lập, không biết bảng videos nên short-video phải tự validate video tồn tại trước và tự thi hành việc ẩn video khi đếm vượt ngưỡng.
- if missing: Không có nơi lưu report chống lặp và đếm distinct reporter: video vi phạm không bao giờ tự bị ẩn, hoặc một người report lặp có thể ép gỡ video.
- evidence: apps/core-api/src/modules/short-video/short-video.service.ts:490
- review: sound — Validate qua getVideoOrThrow trước, reportVideo trả distinctReporterCount, tự ẩn khi >= ngưỡng; report không trừ trust (trustPenaltyApplied 0).

### author PublicProfileDto (c3) with user [service, out, di-call]

- endpoints: e2, e3, e4, e9, e10
- what: UserService.getByIdOrThrow/findByIds dựng author PublicProfileDto cho video và comment, batch một lần cho cả trang.
- why: Hồ sơ do User sở hữu; PublicProfileDto là DTO công khai duy nhất để không lộ birth date/region/trust score, và batch-load để web không gọi GET /users/:id cho từng video/comment.
- if missing: Web phải gọi N request lấy tác giả, hoặc short-video tự dựng DTO hồ sơ có nguy cơ lộ field riêng tư.
- evidence: apps/core-api/src/modules/short-video/short-video.controller.ts:209, apps/core-api/src/modules/short-video/short-video.controller.ts:216
- review: sound — getAuthor và getAuthors batch bằng findByIds, PublicProfileDto không chứa birth date/trust.

### videos/views/reactions (c4) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13
- what: Ghi/đọc videos, video_views, video_reactions, video_comments; chuyển trạng thái bằng conditional UPDATE ... WHERE status = 'từ'; counter denormalized tăng giảm trong transaction (riêng xoá comment không kiểm affected, view qualified không có guard); hai job quét/ghi hàng loạt videos bằng SQL.
- why: Không dùng SELECT FOR UPDATE: conditional UPDATE (thua race = 0 rows) chặn double-transition. Unique (videoId,viewerId)/(videoId,userId)/idempotencyKey chặn trùng HÀNG view/like và upload trùng; nhưng viewCount (đọc view.qualified rồi increment) và giảm commentCount khi xoá comment không có guard nên request song song có thể đếm lệch.
- if missing: Hai request đồng thời có thể finalize/duyệt cùng video hai lần, đếm view/like đôi và tạo video trùng khi retry upload.
- evidence: apps/core-api/src/modules/short-video/entities/video-comment.entity.ts:10, apps/core-api/src/modules/short-video/entities/video-reaction.entity.ts:10, apps/core-api/src/modules/short-video/entities/video-view.entity.ts:15
- review: corrected — Unique chỉ chặn trùng hàng view/like; viewCount không có guard so sánh-đặt và xoá comment giảm commentCount không kiểm affected.

### duyệt/gỡ video + audit (c5) with admin [service, in, di-call]

- endpoints: 
- what: AdminService gọi ShortVideoService.listPendingReview/listPublished/adminApprove/adminReject/adminRemove, truyền manager để chuyển trạng thái video và ghi audit log trong CÙNG transaction.
- why: Với VIDEO_MODERATION_MODE=pre video chỉ lên published khi admin duyệt; quyết định kiểm duyệt phải atomic với audit log (cùng pattern banUser) và chỉ short-video được ghi videos.status.
- if missing: Video kẹt pending_review mãi không lên được; hoặc hành động admin không có audit hay audit lệch với trạng thái thật.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:151, apps/core-api/src/modules/admin/admin.service.ts:157, apps/core-api/src/modules/admin/admin.service.ts:162
- review: sound — approveVideo/reject/remove chạy trong dataSource.transaction cùng auditLogService.record; default VIDEO_MODERATION_MODE là pre. Đúng.

### guard video cho tặng quà (c6) with gift [service, in, di-call]

- endpoints: 
- what: GiftService.sendVideoGift gọi ShortVideoService.getVideoOrThrow(user, videoId) rồi lấy video.authorUserId làm người nhận quà.
- why: Người nhận suy từ video chứ client không gửi receiverUserId (chống gửi lệch người); và chỉ video mà người tặng nhìn thấy được (published hoặc của chính họ) mới được tặng, guard đó thuộc short-video.
- if missing: Client có thể tặng quà cho video đã removed/chưa duyệt hoặc chỉ định người nhận tuỳ ý.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:280
- review: sound — sendVideoGift gọi getVideoOrThrow (tác giả hoặc published) rồi lấy authorUserId làm người nhận, chặn tự tặng.

### presigned PUT + dọn object (c50) with external:object-storage [external, out, sdk]

- endpoints: e1, e13
- what: VideoStoragePort: generateStorageKey(authorId), issueUploadUrl(storageKey) trả presigned URL cho client PUT thẳng body video, delete(storageKey) dọn object mồ côi. Hiện chỉ có DevVideoStorageAdapter (URL giả) và UnavailableVideoStorageAdapter cho production.
- why: Body video lớn không được đi qua NestJS nên storage ngoài phải giữ file; tách generateStorageKey (thuần) khỏi issueUploadUrl (I/O) để replay Idempotency-Key cấp lại ĐÚNG URL của storageKey đã tạo. Upload bỏ dở/failed để lại object mồ côi nên sweeper phải gọi delete.
- if missing: Phải nhận binary video qua core-api (nghẽn băng thông, DoS) hoặc upload bỏ dở để lại object mồ côi tốn dung lượng không ai dọn.
- evidence: apps/core-api/src/modules/short-video/ports/video-storage.port.ts:12, apps/core-api/src/modules/short-video/ports/video-storage.port.ts:16
- review: sound — Port tách generateStorageKey và issueUploadUrl; factory chọn Dev hoặc Unavailable. Sweeper xoá object của MỌI video failed mỗi tick (xoá idempotent, không có cờ đã dọn).

### transcode sau upload (c51) with external:video-transcode [external, out, sdk]

- endpoints: e2
- what: VideoTranscodePort.transcode(storageKey) trả {playbackUrl, thumbnailUrl, durationSeconds}; finalizeUpload lưu 3 giá trị này vào Video rồi chuyển pending_review|published. Hiện chỉ có Dev adapter đồng bộ; vendor thật (Cloudflare Stream/Mux) chưa được nối.
- why: Video client upload là file thô; phải transcode ra playback/thumbnail/duration thì mới phát được. Vendor thật sẽ bất đồng bộ (webhook) nên 'processing' là trạng thái trung gian riêng, và lỗi transcode phải đẩy video sang failed thay vì để kẹt.
- if missing: Video không có playbackUrl/thumbnail/duration nên không phát được, và không có điểm nào để phát hiện file hỏng trước khi lên published.
- evidence: apps/core-api/src/modules/short-video/ports/video-transcode.port.ts:14
- review: sound — finalizeUpload transition uploading sang processing, transcode, update có điều kiện sang pending_review/published; lỗi chuyển failed. Đúng.

### reel feed + gửi watch-time (c52) with web [client, in, http]

- endpoints: e3, e6, e7, e10
- what: Web gọi GET /videos (scope for_you|following), POST /videos/{id}/views với watchTimeMs khi người xem xem video, tim/bỏ tim và comment; chưa có UI upload nào gọi upload-intent/finalize.
- why: Server không biết người xem đã xem bao lâu: chỉ client đo watchTimeMs và báo lên để viewCount chỉ tăng khi qualified. Không có realtime event cho video nên web phải refetch REST để thấy số liệu mới.
- if missing: viewCount (đầu vào của rankScore) không bao giờ tăng, ranking đứng yên và danh sách không tự cập nhật.
- evidence: apps/web/src/features/short-video/api.ts:137
- review: sound — Web chỉ gọi views/reactions/comments và list; không có UI gọi upload-intent/finalize trong apps/web.

### hàng chờ duyệt video (c53) with admin-console [client, in, http]

- endpoints: 
- what: Admin console gọi GET /api/v1/admin/videos/pending|published và POST approve|reject|remove (route ở AdminController, rồi AdminService gọi vào ShortVideoService qua c5).
- why: Mặc định VIDEO_MODERATION_MODE=pre: video chỉ public sau khi người kiểm duyệt bấm duyệt trong console; không có UI này thì không ai chuyển được pending_review sang published.
- if missing: Video upload xong nằm vĩnh viễn ở pending_review, và video bị report dưới ngưỡng auto-hide không có cách gỡ thủ công.
- evidence: apps/admin/src/features/moderation/videos-api.ts:25
- review: sound — videos-api.ts có pending, published, approve, reject, remove; comment ở đầu file nói không có list published là cũ.

## Notes

- Code chỉ có PORT + adapter dev/unavailable: DevVideoStorageAdapter trả URL giả https://dev-storage.invalid/..., production dùng UnavailableVideo*Adapter, và nếu VIDEO_UPLOAD_ENABLED=true dưới NODE_ENV=production thì factory ném lỗi lúc boot. Nghĩa là KHÔNG có object storage/transcode vendor thật nào được nối; hai cạnh external bên dưới mô tả hợp đồng của port, không phải tích hợp đang chạy.
- VideoStoragePort.getPlaybackUrl/getThumbnailUrl khai báo trong port nhưng ShortVideoService không gọi: URL playback/thumbnail lấy từ TranscodeResult do VideoTranscodePort.transcode trả về.
- Khác Feed, short-video KHÔNG lọc block: getVideoOrThrow/listPublished không gọi SafetyService.isBlocked/getBlockedUserIds, nên người đã block nhau vẫn thấy/tim/bình luận video của nhau. Doc short-video-service.md không hứa lọc block nhưng đây là điểm khác với Feed cần chú ý. Guest cũng không bị chặn tạo like/comment ở module này (không có assertNotGuest).
- Doc § 2 nói finalize gọi transcode đồng bộ trong một request (Dev port); code đúng vậy: lỗi transcode chỉ chuyển processing -> failed rồi trả video, và VideoSweeperService mới gọi storage.delete dọn object mồ côi.
- Tất cả route /videos là jwt qua APP_GUARD JwtAuthGuard toàn cục; không @Public, không roles. Các route /admin/videos/* nằm ở AdminController (RequireAdminPermission ResolveReports), không phải ở module này.
- Gift (POST /videos/:videoId/gifts) và Admin gọi vào ShortVideoService qua DI nên không map vào endpoint REST nào của module này; Gift/Admin sở hữu route đó.
