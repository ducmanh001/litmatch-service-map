# Service map: Notification (+ Web Push)

id: service-map:notification
module: module:notification
deployable: core-api
confidence: high

Thông báo in-app và Web Push

Sở hữu bảng notifications (in-app, cursor theo seq) và web_push_subscriptions; là điểm duy nhất các module nghiệp vụ gọi để ghi thông báo (createWithManager/create) rồi đẩy push best-effort (sendPush). Không sở hữu nội dung hiển thị (payload tối thiểu, client tự render), không có realtime event riêng và không publish Redis.

Owns: bảng notifications (userId, type, payload jsonb, readAt, seq); bảng web_push_subscriptions (endpoint unique, p256dh, auth)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/notifications | jwt | Liệt kê notification của chính user, mới nhất trước, phân trang cursor theo seq (cursor sai trả 400 CURSOR_INVALID). |
| e2 | rest | GET | /api/v1/notifications/unread-count | jwt | Đếm notification chưa đọc (readAt IS NULL) cho badge chuông; web poll mỗi 60s. |
| e3 | rest | POST | /api/v1/notifications/web-push/subscription | jwt | Đăng ký/cập nhật browser push subscription (endpoint+p256dh+auth) của user; upsert theo endpoint unique. |
| e4 | rest | DELETE | /api/v1/notifications/web-push/subscription | jwt | Huỷ subscription theo endpoint, chỉ xoá hàng thuộc chính user (delete theo userId+endpoint). |
| e5 | rest | POST | /api/v1/notifications/:notificationId/read | jwt | Đánh dấu đã đọc, idempotent; chỉ chủ sở hữu (người khác nhận 404 NOTIFICATION_NOT_FOUND để không lộ sự tồn tại). |

## Coordination edges (why this service talks to others)

### ghi notification + push sub (c1) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5
- what: Đọc/ghi bảng notifications (save, query theo seq, update readAt) và web_push_subscriptions (upsert theo endpoint, find theo userId, delete).
- why: Hàng notification là nguồn sự thật của thông báo (push chỉ là best-effort) nên phải bền vững; createWithManager nhận EntityManager của caller để hàng này commit/rollback cùng hành động gốc (vd gift/like/comment/match) trong MỘT transaction.
- if missing: Mất thông báo khi push fail hoặc user offline; nếu ghi ngoài transaction của caller thì có thể có thông báo 'ma' cho hành động đã rollback, hoặc hành động thành công mà không có thông báo.
- evidence: apps/core-api/src/modules/notification/entities/notification.entity.ts:26, apps/core-api/src/modules/notification/entities/web-push-subscription.entity.ts:11
- review: sound — createWithManager nhận EntityManager của caller; WebPushSubscriptionService.upsert theo ['endpoint'] với unique index; markRead update readAt. Đã xác nhận.

### broadcast admin + push (c2) with admin [service, in, di-call]

- endpoints: e1, e2
- what: AdminService.broadcastNotification gọi createBroadcastWithManager(manager, recipientIds, {title, body}) trong transaction chung với auditLogService.record, rồi sendPush cho từng notification sau commit.
- why: Broadcast phải ghi audit 'notification.broadcast.sent' atomic với các hàng notification (chunk 500/INSERT, cùng manager); push chỉ chạy sau commit để không gửi thông báo cho broadcast bị rollback.
- if missing: Admin gửi được broadcast mà không có audit (hoặc audit mà không có thông báo); push có thể đi trước khi hàng DB tồn tại.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:438, apps/core-api/src/modules/admin/admin.service.ts:463
- review: sound — createBroadcastWithManager chunk 500 và auditLogService.record cùng manager; sendPush chạy sau transaction. Đúng.

### like/comment notify (c3) with feed [service, in, di-call]

- endpoints: e1, e2
- what: FeedService.like/createComment gọi createWithManager(manager, {type post_liked|post_commented, payload {postId, commentId?, actorUserId}}) trong transaction của hành động, rồi sendPush sau commit; bỏ qua nếu actor = author.
- why: Thông báo phải atomic với việc tăng commentCount/ghi like: không được có thông báo cho comment đã rollback. Chỉ có 1 consumer nên gọi DI trực tiếp, không cần Outbox/Kafka (doc § 1).
- if missing: Tác giả bài viết không biết có like/comment, hoặc nhận thông báo trỏ tới comment không tồn tại.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:260, apps/core-api/src/modules/feed/feed.service.ts:275, apps/core-api/src/modules/feed/feed.service.ts:360
- review: sound — createWithManager trong transaction comment/like, bỏ qua tự like/comment, sendPush sau commit. Đúng.

### tin nhắn + cảnh báo streak (c4) with friend [service, in, di-call]

- endpoints: e1, e2
- what: FriendService.sendMessage gọi create({friend_message, payload {conversationId, senderUserId, preview}}) rồi sendPush; streak-warning.job cũng gọi create để nhắc streak sắp mất.
- why: Tin nhắn Friend đã commit transaction riêng nên Notification phải tự mở transaction (create, không createWithManager) và chạy NGAY SAU khi message persist, bọc try/catch để lỗi thông báo không làm fail luồng gửi tin nhắn đã thành công.
- if missing: Người nhận offline không thấy badge/push cho tin nhắn mới; nếu dùng chung transaction thì lỗi thông báo có thể rollback cả tin nhắn.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:440, apps/core-api/src/modules/friend/friend.service.ts:449, apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:106
- review: sound — create dùng notificationRepo.manager (ghi độc lập), bọc try/catch; job streak cũng gọi create + sendPush.

### thông báo nhận quà (c5) with gift [service, in, di-call]

- endpoints: e1, e2
- what: GiftService.sendGift gọi createWithManager(manager, {gift_received, payload {roomId, senderUserId, giftCode, priceDiamond}}) bên trong withinTransaction cùng bút toán Economy; sendPush chỉ khi !replayed.
- why: Quà là giao dịch tiền: thông báo phải commit/rollback cùng GiftEvent và ledger trong MỘT transaction; replay Idempotency-Key không được tạo hoặc push thông báo thứ hai.
- if missing: Người nhận quà không được báo; hoặc retry cùng Idempotency-Key tạo thông báo trùng; hoặc có thông báo cho lần gửi quà đã rollback vì thiếu kim cương.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:238, apps/core-api/src/modules/gift/gift.service.ts:261, apps/core-api/src/modules/gift/gift.service.ts:335
- review: sound — withinTransaction chỉ chạy khi không replay (ledger trả replayed trước đó); sendPush chỉ khi !replayed. Đúng.

### thông báo match_confirmed (c6) with matching [service, in, di-call]

- endpoints: e1, e2
- what: matcher-worker và MatchingService.confirmTicket gọi createWithManager(manager, {match_confirmed, payload {sessionId, matchType}}) trong transaction chốt cặp, rồi sendPush cho 2 người sau commit.
- why: Thông báo phải atomic với việc chốt MatchSession/ticket để cả hai người cùng nhận hoặc cùng không; payload TUYỆT ĐỐI không chứa partnerId vì Soul Match ẩn danh tới khi unlock (docs/06).
- if missing: Người dùng đóng app không biết đã match; nếu lộ partnerId trong payload thì phá vỡ ẩn danh Soul Match.
- evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:299, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:387, apps/core-api/src/modules/matching/matching.service.ts:381
- review: sound — Payload chỉ {sessionId, matchType}, không có partnerId; sendPush sau transaction ở cả matcher-worker và confirmTicket.

### gửi Web Push (VAPID) (c50) with external:web-push [external, out, sdk]

- endpoints: e3, e4
- what: WebPushNotificationAdapter.send gọi webpush.sendNotification(subscription, JSON {notificationId, type, payload}, TTL 3600) tới endpoint của push service trình duyệt; 404/410 trả removeSubscription để xoá subscription chết.
- why: Trình duyệt đóng tab chỉ nhận được thông báo qua push service của hãng (FCM/Mozilla/Apple) ký bằng khoá VAPID; web_push_subscriptions lưu endpoint để core-api gửi tới đúng trình duyệt, và phải dọn endpoint 404/410 để không gửi mãi vào endpoint đã huỷ.
- if missing: User đóng tab không nhận được thông báo nào ngoài việc mở lại app và poll; endpoint chết tích luỹ và mỗi sendPush tốn thêm lời gọi mạng vô ích.
- evidence: apps/core-api/src/common/platform/web-push-notification.adapter.ts:46, apps/core-api/src/common/platform/web-push-notification.adapter.ts:63
- review: sound — setVapidDetails, sendNotification TTL 3600, 404/410 thành removeSubscription và sendPush gọi removeById. Đúng.

### push FCM (chưa có token) (c51) with external:fcm [external, out, http]

- endpoints: 
- what: FcmPushNotificationAdapter.send POST FCM HTTP v1 messages:send với {token, data.notification_type}, xác thực bằng access token OAuth từ service account; chỉ chạy khi NOTIFICATION_PUSH_PROVIDER=fcm VÀ caller truyền deviceToken.
- why: Suy luận: đây là kênh push native Android theo Port/Adapter để business code chỉ phụ thuộc PushNotificationPort; hiện không caller nào truyền deviceToken nên adapter luôn trả 'skipped' (xem notes).
- if missing: Không có kênh native Android khi sau này có đăng ký device token; hiện tại không có hành vi nào mất vì chưa có token.
- evidence: apps/core-api/src/common/platform/fcm-push-notification.adapter.ts:36, apps/core-api/src/common/platform/fcm-push-notification.adapter.ts:154
- review: sound — Không caller nào truyền deviceToken vào sendPush nên adapter luôn trả skipped; HTTP v1 messages:send đúng.

### push APNs (chưa có token) (c52) with external:apns [external, out, http]

- endpoints: 
- what: ApnsPushNotificationAdapter mở kết nối node:http2 tới APNs gửi thông báo cho device token iOS; chỉ được chọn khi NOTIFICATION_PUSH_PROVIDER=apns và cần deviceToken từ caller.
- why: Suy luận: kênh push native iOS đặt sau cùng PushNotificationPort với FCM, chọn bằng provider-factory; cùng giới hạn là chưa có luồng đăng ký device token nên hiện chưa bao giờ gửi thật.
- if missing: Không có kênh native iOS khi sau này có token; hiện tại không mất hành vi nào.
- evidence: apps/core-api/src/common/platform/apns-push-notification.adapter.ts:42
- review: sound — Dùng connect từ node:http2, JWT ES256 ký bằng jose; chỉ chạy khi provider=apns và có deviceToken, hiện chưa caller nào có token.

### service worker + subscribe (c53) with web [client, both, sdk]

- endpoints: e3, e4
- what: apps/web đăng ký /web-push/sw.js, gọi pushManager.subscribe với NEXT_PUBLIC_WEB_PUSH_PUBLIC_KEY rồi POST subscription lên e3; sw.js nhận sự kiện push, chỉ hiện thông báo khi không có tab visible và mở /chat/:senderUserId khi click.
- why: Push tới khi tab đã đóng chỉ xử lý được bởi service worker phía trình duyệt; applicationServerKey phía web phải khớp WEB_PUSH_PUBLIC_KEY của core-api (cùng cặp VAPID) nếu không push service từ chối. Tab đang mở thì đã có toast/realtime nên sw im lặng để khỏi trùng.
- if missing: Không có subscription để lưu, hoặc khoá VAPID lệch nên mọi push bị từ chối; không service worker thì push đến mà không hiển thị.
- evidence: apps/web/src/features/notification/web-push.ts:70, apps/web/public/web-push/sw.js:2
- review: sound — sw.js chỉ hiện khi không có cửa sổ visible và mở /chat/:senderUserId cho friend_message; applicationServerKey lấy từ NEXT_PUBLIC_WEB_PUSH_PUBLIC_KEY.

### poll badge + danh sách (c54) with web [client, in, http]

- endpoints: e1, e2, e5
- what: Web gọi GET /notifications/unread-count mỗi 60s (refetchInterval), tải GET /notifications khi mở chuông và POST /:id/read; invalidate cache khi nhận event friend_message.
- why: Notification không phát realtime event riêng và push là best-effort, nên REST poll là kênh chính để badge không lệch; endpoint chỉ đọc theo userId từ JWT nên mỗi user chỉ thấy thông báo của mình.
- if missing: Badge chuông không bao giờ cập nhật khi push thất bại hoặc tab không nhận realtime; user không đọc/đánh dấu được thông báo.
- evidence: apps/web/src/features/notification/api.ts:54
- review: sound — Poll unread-count mỗi 60_000ms; bridge invalidate notificationKeys.all khi nhận friend.message. Đúng.

## Notes

- Code vs doc: doc nói FCM/APNs nhận device token từ caller, nhưng KHÔNG caller nào truyền deviceToken vào sendPush (admin/feed/friend/gift/matching đều gọi sendPush(notification)); FcmPushNotificationAdapter.send trả 'skipped' khi thiếu deviceToken. Vậy FCM/APNs hiện chỉ là hạ tầng chưa có luồng đăng ký token native (doc § 7 cũng ghi ngoài scope); push thật duy nhất hiện nay là Web Push.
- NotificationService.trackAnalytics (AnalyticsPort/PostHog) không có caller nào trong src (đã grep) nên không thêm collaborator PostHog; doc § 5 mô tả adapter nhưng chưa có luồng nào bắn event qua module này.
- Notification không publish realtime event riêng: badge chuông được cập nhật nhờ poll /notifications/unread-count mỗi 60s và nhờ RealtimeNotificationBridge nghe event friend_message do module Friend phát.
- Friend.sendMessage bỏ qua cả in-app lẫn push khi người nhận mute hội thoại (friend.service.ts:439) - quyết định nằm ở caller, Notification không biết trạng thái mute.
- Web Push chỉ hoạt động khi WEB_PUSH_ENABLED=true (mặc định tắt); khi tắt adapter trả 'skipped' và chỉ còn in-app + poll.
