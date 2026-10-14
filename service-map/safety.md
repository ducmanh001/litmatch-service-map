# Service map: Safety (Trust & Safety)

id: service-map:safety
module: module:safety
deployable: core-api
confidence: high

Report, block và luật ẩn người dùng

Sở hữu log append-only Report/Block, trust-score penalty chống vote-kick và các câu hỏi đồng bộ 'hai người này có được thấy/ghép/nhắn nhau không' (isBlocked, getBlockedUserIds, getHiddenUserIds, canPair). KHÔNG sở hữu ban tài khoản (UserStatus.Banned nằm ở User/Admin), không sở hữu video/feed (module gọi tự validate target) và không áp penalty lên hàng đợi matching (Matching tự đọc trustScore).

Owns: bảng reports (append-only; chỉ cột status pending|resolved|dismissed được sửa bởi moderator; targetType user|video); bảng blocks (log action blocked|unblocked; trạng thái = dòng mới nhất theo cặp); unique index uq_reports_video_reporter (1 người report 1 video 1 lần)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/safety/reports | jwt | Report 1 user: chặn tự report, khoá dòng users của target rồi ghi Report + trừ trust score (1 report hiệu lực/cặp/SAFETY_REPORT_COOLDOWN_DAYS, trần SAFETY_TRUST_PENALTY_DAILY_CAP/ngày) trong CÙNG transaction; throttle 10 report/giờ/tài khoản. |
| e2 | rest | POST | /api/v1/safety/blocks/:targetUserId | jwt | Block target (idempotent): chặn tự block, xác nhận target tồn tại, ghi dòng action=blocked nếu dòng mới nhất chưa là blocked; trả 204. |
| e3 | rest | DELETE | /api/v1/safety/blocks/:targetUserId | jwt | Unblock target (idempotent): chỉ ghi dòng action=unblocked khi dòng mới nhất đang là blocked; trả 204. |
| e4 | rest | GET | /api/v1/safety/blocks/:targetUserId | jwt | Cho caller biết MÌNH có đang block target hay không (một chiều, không lộ việc bị block ngược). |

## Coordination edges (why this service talks to others)

### kiểm tra target, trừ điểm (c1) with user [service, out, di-call]

- endpoints: e1, e2
- what: UserService.getByIdOrThrow(targetUserId) trước khi report/block; UserService.adjustTrustScore(manager, targetUserId, -penalty, floor) với manager của transaction report.
- why: trustScore là cột của User nên chỉ User được sửa; Safety phải truyền EntityManager của mình để Report và điểm trừ commit/rollback cùng nhau (không lệch nếu một bước lỗi). getByIdOrThrow chặn report/block vào id không tồn tại.
- if missing: Có thể ghi Report/Block cho user không tồn tại; nếu trừ điểm ở transaction riêng thì Report có mà điểm không trừ (hoặc ngược lại) khi một bước fail.
- evidence: apps/core-api/src/modules/safety/safety.service.ts:66, apps/core-api/src/modules/safety/safety.service.ts:128, apps/core-api/src/modules/safety/safety.service.ts:199
- review: sound — getByIdOrThrow chạy trước transaction; adjustTrustScore nhận manager của transaction report — đúng.

### khoá dòng users của target (c2) with user [service, out, shared-db]

- endpoints: e1
- what: manager.getRepository(User).findOne({ lock: pessimistic_write }) trên dòng users của target ngay đầu transaction report().
- why: Cooldown theo cặp và trần penalty/ngày được tính bằng cách đọc tổng đã áp rồi mới ghi. Hai report đồng thời của nhiều reporter sẽ cùng đọc tổng cũ và vượt trần, nên phải tuần tự hoá theo target trên dòng users bền vững (comment safety.service.ts:84-86).
- if missing: Report dồn dập đồng thời vượt SAFETY_TRUST_PENALTY_DAILY_CAP hoặc trừ trùng trong cùng cooldown, cho phép vote-kick đánh sập trust score.
- evidence: apps/core-api/src/modules/safety/safety.service.ts:87
- review: sound — findOne pessimistic_write trên User target ở đầu transaction trước khi đọc tổng penalty/cooldown — đúng comment 84-86.

### reports/blocks append-only (c3) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4
- what: Ghi và truy vấn reports/blocks; trạng thái block = dòng mới nhất theo cặp (findOne ORDER BY createdAt DESC, DISTINCT ON cho tập 2 chiều); canPair đọc report/block trong cửa sổ SAFETY_REMATCH_COOLDOWN_DAYS, còn getHiddenUserIds lấy mọi report từng có (không theo cooldown).
- why: Log append-only giữ toàn bộ lịch sử block/unblock/report cho điều tra T&S (docs/06: hành động nhạy cảm không được xoá), nên 'đang block' phải suy từ dòng mới nhất thay vì sửa một dòng trạng thái.
- if missing: Mất lịch sử điều tra, không có nguồn sự thật chung để Feed, Friend, Discovery, Matching cùng kiểm tra cùng một luật block/report.
- evidence: apps/core-api/src/modules/safety/entities/block.entity.ts:20, apps/core-api/src/modules/safety/entities/report.entity.ts:42
- review: corrected — getHiddenUserIds đọc mọi report của cặp, không giới hạn cửa sổ cooldown; chỉ canPair dùng cooldown.

### queue + đổi status report (c4) with admin [service, in, di-call]

- endpoints: 
- what: AdminService.listReports gọi SafetyService.findReportsPage(filter, limit, offset); resolveReport/dismissReport gọi setReportStatus(manager, reportId, status) trong cùng transaction với auditLogService.record.
- why: Chỉ Safety sở hữu bảng reports nên admin phải đi qua nó; nhận manager để đổi status và audit log commit cùng nhau (hành động nhạy cảm không được thành công một nửa, cùng pattern UserService.banUser). Chỉ cột status được mutate.
- if missing: Moderator không có queue để xem report; hoặc resolve/dismiss mà thiếu audit log, hoặc admin sửa trực tiếp bảng của Safety.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:112, apps/core-api/src/modules/admin/admin.service.ts:129
- review: sound — AdminService.setReportStatus mở dataSource.transaction, gọi safetyService.setReportStatus(manager) rồi auditLogService.record — đúng.

### lọc người bị ẩn (discovery) (c5) with discovery [service, in, di-call]

- endpoints: 
- what: DiscoveryService.browse và NearbyService gọi SafetyService.getHiddenUserIds(userId) rồi nối vào excludeUserIds của query ứng viên.
- why: Discovery chỉ biết thông tin hồ sơ; tập 'đã block 2 chiều hoặc từng report nhau' do Safety giữ. Phải lấy tập ẩn một lần cho cả trang rồi loại ở query (gọi từng ứng viên sẽ N+1); report ẩn vĩnh viễn vì reports là append-only.
- if missing: Người dùng vẫn thấy người họ vừa block/report (hoặc người đã block họ) trong danh sách duyệt và nearby, kể cả sau khi bấm block.
- evidence: apps/core-api/src/modules/discovery/discovery.service.ts:49, apps/core-api/src/modules/discovery/nearby.service.ts:174
- review: sound — Hidden-set = block 2 chiều + mọi report 2 chiều, vĩnh viễn (comment getHiddenUserIds); discovery/nearby lấy một lần cho cả trang.

### loại tác giả bị block (c6) with feed [service, in, di-call]

- endpoints: 
- what: FeedService.listFeed gọi getBlockedUserIds(viewer) để NOT IN authorUserId; listUserTimeline gọi isBlocked 2 chiều rồi trả rỗng; StoryService (feed/services/story.service.ts) cũng lọc story, kiểm isBlocked 2 chiều khi xem và lọc người xem story theo getBlockedUserIds.
- why: Bài đăng thuộc Feed nhưng quan hệ block thuộc Safety. Feed lọc cả trang bằng một query tập id và với timeline trả rỗng (không throw) để không lộ trạng thái block bằng hành vi khác với 'chưa đăng bài' (oracle-safe).
- if missing: Bài của người đã block (hoặc người block mình) vẫn hiện trong feed và timeline, hoặc lỗi khác biệt làm lộ ai block ai.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:103, apps/core-api/src/modules/feed/feed.service.ts:150, apps/core-api/src/modules/feed/feed.service.ts:151
- review: sound — getBlockedUserIds 2 chiều; story.service.ts dùng cả isBlocked 2 chiều và getBlockedUserIds như mô tả.

### chặn chat/gọi khi bị block (c7) with friend [service, in, di-call]

- endpoints: 
- what: FriendService.sendMessage gọi isBlocked(userId, partnerId) và isBlocked(partnerId, userId), nếu block trả CONVERSATION_NOT_FOUND 404; ProfileSocialService.canCall/assertNotBlocked kiểm block 2 chiều ngay trước khi mint token cuộc gọi/xem hành động hồ sơ.
- why: Block phải có hiệu lực ngay ở thời điểm GỬI/GỌI chứ không chỉ khi mở conversation: Friend giữ conversation nên phải hỏi Safety mỗi lần, và isBlocked chỉ một chiều nên phải gọi hai lần. Trả cùng mã lỗi với 'không phải thành viên' để không có oracle.
- if missing: Người bị block vẫn nhắn hoặc gọi được cho người đã block họ, hoặc mã lỗi khác biệt tiết lộ ai block ai.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:346, apps/core-api/src/modules/friend/friend.service.ts:347, apps/core-api/src/modules/friend/services/profile-social.service.ts:152
- review: sound — friend.service.ts gọi isBlocked hai chiều rồi CONVERSATION_NOT_FOUND 404; canCall/assertNotBlocked kiểm 2 chiều.

### canPair khi ghép; ẩn lời mời (c8) with matching [service, in, di-call]

- endpoints: 
- what: Matching gọi SafetyService.canPair(a, b) qua token MATCH_INTERACTION_POLICY (useExisting SafetyService) trong tryPair của matcher-worker và khi accept invite; invite.service còn gọi getHiddenUserIds khi tạo lời mời và khi đọc inbox.
- why: Block/report có thể phát sinh sau khi ticket vào hàng đợi hoặc lời mời được tạo (docs/10 § 10.0.C), nên chốt phải kiểm lại ĐÚNG lúc ghép/accept, không chỉ lúc enqueue. Lời gọi canPair được gọi khi transaction của Matching đang mở nhưng chạy trên kết nối riêng của Safety; kết quả quyết định có tạo MatchSession hay không.
- if missing: Hai người vừa block hoặc report nhau vẫn bị ghép/accept nếu một trong hai block sau lúc vào hàng đợi; inbox hiển thị lời mời từ người đã ẩn.
- evidence: apps/core-api/src/modules/matching/services/invite.service.ts:199, apps/core-api/src/modules/matching/services/invite.service.ts:581
- review: corrected — canPair không chạy trong transaction của Matching: không nhận manager, dùng repo của Safety (kết nối riêng).

### ẩn mood của người bị block (c9) with mood [service, in, di-call]

- endpoints: 
- what: MoodService.getPublicMood gọi getBlockedUserIds(viewerId) và trả null nếu targetId có trong tập (viewer khác target).
- why: Mood hiển thị trên hồ sơ/danh sách bạn; block 2 chiều phải che nó, và phải kiểm lại mỗi lần đọc (không cache) để unblock/block có hiệu lực ngay (docs/10 § 10.0.C). Chỉ xét block, không xét report.
- if missing: Người bị block vẫn thấy mood hiện tại của người block họ.
- evidence: apps/core-api/src/modules/mood/mood.service.ts:131
- review: sound — getPublicMood chỉ gọi getBlockedUserIds, không xét report, không cache — đúng.

### canPair trong ghép movie (c10) with movie-match [service, in, di-call]

- endpoints: 
- what: MovieMatchService duyệt ứng viên trong hàng đợi và gọi SafetyService.canPair(userId, candidate.user_id) bên trong transaction ghép; false thì thử ứng viên kế tiếp.
- why: Movie-match có hàng đợi riêng (không đi qua Matching) nên tự phải kiểm block/report ĐÚNG lúc ghép, không tin trạng thái lúc enqueue. Lời gọi đồng bộ vì nó quyết định ứng viên nào thành đối tác của session.
- if missing: Hai người đã block/report nhau vẫn bị ghép xem phim cùng nhau.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:338
- review: sound — canPair gọi trong vòng lặp ứng viên bên trong transaction ghép, false thì thử người kế (movie-match.service.ts:338).

### canPair trong ghép palm (c11) with palm-match [service, in, di-call]

- endpoints: 
- what: PalmMatchService duyệt ứng viên hàng đợi và gọi SafetyService.canPair(userId, candidate.user_id) trong transaction ghép, bỏ qua ứng viên bị từ chối.
- why: Tương tự movie-match: hàng đợi palm tự quản nên phải hỏi Safety đúng lúc ghép; cặp bị block active hoặc có report/block trong SAFETY_REMATCH_COOLDOWN_DAYS không được gặp lại.
- if missing: Cặp đã block/report nhau vẫn được ghép palm match lại.
- evidence: apps/core-api/src/modules/palm-match/palm-match.service.ts:158
- review: sound — palm-match tương tự; canPair gồm block active và report/block trong SAFETY_REMATCH_COOLDOWN_DAYS đúng code.

### ghi report video, đếm (c12) with short-video [service, in, di-call]

- endpoints: 
- what: ShortVideoService.reportVideo gọi SafetyService.reportVideo(reporterId, videoId, reason, description) và nhận distinctReporterCount để so VIDEO_REPORT_AUTOHIDE_THRESHOLD rồi tự ẩn video.
- why: Report nằm chung bảng reports (targetType=video) với unique (reporter, video); Safety trung lập, không biết bảng videos và không trừ trust score nên short-video phải tự validate video và tự quyết auto-hide dựa trên số reporter Safety trả về.
- if missing: Short-video phải tự dựng bảng report riêng, hoặc không có đếm distinct reporter nên không auto-hide được video bị report nhiều.
- evidence: apps/core-api/src/modules/short-video/short-video.service.ts:490
- review: sound — reportVideo không trừ trust score, bắt unique violation rồi trả distinctReporterCount; short-video tự so ngưỡng.

## Notes

- Doc § 1 nói Report 'không có cột trạng thái mutable ở GĐ4'; code đã có Report.status (pending|resolved|dismissed) và SafetyService.setReportStatus do AdminService gọi trong cùng transaction với audit log (admin.service.ts:129). Doc § 5 'chưa có admin review' đã lỗi thời.
- Doc § 1/§ 2 chỉ mô tả report user; code còn có Report.targetType=video, targetVideoId và SafetyService.reportVideo (short-video gọi) KHÔNG trừ trust score, trả distinctReporterCount để short-video tự quyết auto-hide.
- Doc § 2 ghi report(): Promise<void> và adjustTrustScore(userId, -delta, reason, manager?); code trả Promise<Report> và UserService.adjustTrustScore(manager, userId, delta, floor) (clamp bằng GREATEST(floor, trust_score + delta) trong SQL).
- SafetyService.getHiddenUserIds (Discovery/Matching invite) không có trong doc: hợp của block active 2 chiều và MỌI report từng tồn tại giữa cặp, vĩnh viễn, bất kể status (kể cả dismissed/resolved) và không theo SAFETY_REPORT_COOLDOWN_DAYS. Khác canPair (có cooldown SAFETY_REMATCH_COOLDOWN_DAYS).
- isBlocked chỉ một chiều; Feed (listUserTimeline), Friend (sendMessage, canCall) phải tự gọi hai lần đảo chiều, còn getBlockedUserIds và canPair đã xét 2 chiều. Dễ sót nếu module mới chỉ gọi một chiều.
- Ràng buộc 'không ghép khi block/report' đi qua token DI MATCH_INTERACTION_POLICY (matching.module.ts:70 useExisting SafetyService; gọi ở matcher-worker.service.ts:277 và invite.service.ts:475) nên AST không thấy lời gọi canPair của Matching; collaborator c8 mô tả cả hai.
- Safety không có kiểm tra ban: user bị Banned chặn ở User/auth, không qua SafetyService. Party-room, calling mint token trực tiếp, gift, soul-match không gọi Safety (chỉ Friend.canCall gián tiếp bảo vệ cuộc gọi); ngoài các module trong danh sách inbound không có nơi nào enforce block.
