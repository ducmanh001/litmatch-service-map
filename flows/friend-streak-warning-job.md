# Job cảnh báo streak sắp mất

id: flow:friend-streak-warning-job
group: Xã hội & nội dung
services: module:friend, module:signaling-gateway, module:notification

Job nền của module friend chạy theo chu kỳ, chỉ một replica được chạy nhờ advisory lock trong Postgres. Nó tìm các conversation sắp mất streak, báo realtime qua Redis và signaling-gateway, tạo thông báo kèm push, rồi đánh dấu đã cảnh báo.

Mỗi tick, friend quét streak trong Postgres, publish realtime qua Redis tới gateway, nhờ notification ghi thông báo rồi đánh dấu đã cảnh báo.

## Participants

- friend (service): friend (StreakWarningJob)
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): signaling-gateway
- notification (service): notification
- web (client): Web client

## Steps

1. friend -> postgres [call/shared-db]: Tick định kỳ: giành advisory lock để chỉ một replica chạy job {apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:62}
2. friend -> postgres [call/shared-db]: Tìm conversation có streak>0, hôm qua đã nhắn, chưa cảnh báo hôm nay (Chỉ chạy khi giờ UTC đã qua STREAK_WARNING_HOURS; giới hạn theo STREAK_WARNING_BATCH_SIZE.) {apps/core-api/src/modules/friend/services/streak.service.ts:141}
3. friend -> postgres [call/shared-db]: Với mỗi conversation: đọc Conversation để lấy 2 thành viên {apps/core-api/src/modules/friend/services/conversation.service.ts:59}
4. friend -> redis [async/redis-pubsub]: Publish friend.streak-at-risk cho từng thành viên (publishRealtimeEvent nuốt lỗi và chỉ log warn.) {apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:100}
5. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
6. signaling-gateway -> web [async/socket]: Emit sự kiện xuống room socket của người dùng {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
7. friend -> notification [call/di-call]: Tạo thông báo StreakAtRisk cho từng thành viên {apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:106}
8. notification -> postgres [call/shared-db]: Ghi bản ghi notification (bảng notifications) (Suy luận: bước ghi DB nằm trong NotificationService.create, không nằm trong trace của friend.) {apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:106}
9. friend -> notification [async/di-call]: Đẩy Web Push cho thông báo vừa tạo (Lỗi create/sendPush của một thành viên bị bắt và chỉ log warn.) {apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:111}
10. friend -> postgres [call/shared-db]: markWarningSent: ghi lastWarningSentAt để hôm nay không cảnh báo lại {apps/core-api/src/modules/friend/services/streak.service.ts:158}

## Invariants

- Job chỉ ghi lastWarningSentAt (markWarningSent) sau khi đã publish realtime và chạy xong vòng tạo thông báo/push, nên mỗi conversation được cảnh báo tối đa một lần mỗi ngày UTC; chỉ lỗi nằm ngoài các try/catch best-effort (ví dụ markWarningSent) mới được thử lại ở tick sau, còn lỗi notification/push bị nuốt thì vẫn được đánh dấu đã cảnh báo.
- friend không ghi currentStreak hoặc longestStreak ở job này; StreakService chỉ trả danh sách conversation và để job gọi Redis và notification, nhờ đó StreakService không phụ thuộc hai module đó.
- Realtime đi qua Redis tới signaling-gateway vì core-api không giữ socket; gateway chỉ relay envelope xuống room user.

## Failure modes

- when Publish Redis cho một thành viên thất bại: Lỗi bị nuốt và log warn; job vẫn tạo thông báo và đánh dấu đã cảnh báo, client còn polling.
- when notification.create hoặc sendPush lỗi cho một thành viên: Bị bắt và log warn rồi bỏ qua; job vẫn gọi markWarningSent cho conversation.
- when Một lỗi khác trong warnOne (ví dụ markWarningSent lỗi): Log error cho conversation đó, chưa đánh dấu nên tick sau thử lại, các conversation còn lại vẫn được xử lý.
- when Conversation bị xoá giữa lúc quét và xử lý: warnOne trả về sớm, bỏ qua êm.

## Endpoints covered

- StreakWarningJob.onApplicationBootstrap [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
