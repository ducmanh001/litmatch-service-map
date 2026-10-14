# Tin nhắn bạn bè: realtime và Web Push

id: flow:message-push-notification
group: Xã hội & nội dung
services: module:friend, module:safety, module:notification, module:signaling-gateway

Một người gửi tin nhắn 1-1; module friend lưu tin, cập nhật streak, rồi báo cho người nhận qua realtime (Redis tới signaling-gateway) và qua thông báo lưu DB kèm Web Push tới trình duyệt đã đăng ký.

Web Push được đăng ký trước ở Notification; sau đó friend lưu tin, phát realtime qua Redis và gateway, rồi nhờ notification ghi thông báo và đẩy Web Push tới các subscription đã đăng ký.

## Participants

- web (client): Web client
- friend (service): Friend
- safety (service): Safety
- notification (service): Notification
- signaling-gateway (service): Signaling gateway
- postgres (infra): PostgreSQL
- redis (infra): Redis
- external:browser-push (external): Dịch vụ Web Push

## Steps

1. web -> notification [call/http]: POST /notifications/web-push/subscription đăng ký browser push subscription (Chạy trước các tin nhắn: đây là cách các dòng subscription mà bước đẩy push đọc về sau được tạo ra.) {apps/core-api/src/modules/notification/notification.controller.ts:65}
2. notification -> postgres [call/shared-db]: Upsert subscription theo endpoint (userId, endpoint, p256dh, auth) (Chỉ ghi bảng subscription của Notification; xung đột trên cột endpoint thì cập nhật dòng cũ.) {apps/core-api/src/modules/notification/services/web-push-subscription.service.ts:20, apps/core-api/src/modules/notification/services/web-push-subscription.service.ts:27}
3. notification -> web [return/http]: Trả 204 No Content
4. web -> friend [call/http]: POST /conversations/:id/messages gửi tin nhắn
5. friend -> safety [call/di-call]: Kiểm tra hai chiều người gửi/nhận có chặn nhau không (Hai lời gọi isBlocked chạy tuần tự (toán tử ||); block bất kỳ chiều nào thì ném 404 CONVERSATION_NOT_FOUND trước khi lưu tin.) {apps/core-api/src/modules/friend/friend.service.ts:346}
6. friend -> postgres [call/shared-db]: Lưu Message và cập nhật Conversation (messageRepo.save rồi conversationRepo.update(lastMessageAt) là hai lệnh riêng, không cùng transaction (lastMessageAt chỉ để sort danh sách). Idempotency-Key trùng cùng conversation và cùng nội dung thì trả lại tin cũ (bỏ qua update); khác nội dung thì ném 409 MESSAGE_IDEMPOTENCY_CONFLICT.) {apps/core-api/src/modules/friend/services/conversation.service.ts:162}
7. friend -> postgres [call/shared-db]: Ghi streak trong transaction riêng (lỗi streak bị bỏ qua, tin vẫn gửi) (recordActivity tự mở transaction khoá row streak; FriendService bọc trong try/catch nên lỗi streak chỉ log error. Đạt mốc thì publish thêm FriendStreakIncreased tới cả hai bên.) {apps/core-api/src/modules/friend/services/streak.service.ts:59}
8. friend -> redis [async/redis-pubsub]: Publish envelope friend.message cho cả người gửi lẫn người nhận {apps/core-api/src/modules/friend/friend.service.ts:425}
9. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
10. signaling-gateway -> web [async/socket]: Relay envelope xuống socket của hai bên đang online {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
11. friend -> notification [call/di-call]: Tạo thông báo in-app cho người nhận {apps/core-api/src/modules/friend/friend.service.ts:440}
12. notification -> postgres [call/shared-db]: Lưu bản ghi Notification in-app {apps/core-api/src/modules/notification/notification.service.ts:60}
13. friend -> notification [call/di-call]: Yêu cầu đẩy push cho thông báo vừa tạo (sendPush chạy sau create; bên trong còn gọi pushPort.send (push chung, lỗi chỉ log warn) trước phần Web Push, không vẽ riêng ở đây.) {apps/core-api/src/modules/friend/friend.service.ts:449}
14. notification -> postgres [call/shared-db]: Đọc các web push subscription của người nhận {apps/core-api/src/modules/notification/services/web-push-subscription.service.ts:36}
15. notification -> external:browser-push [async/sdk]: Gửi Web Push tới từng subscription {apps/core-api/src/modules/notification/notification.service.ts:124}
16. notification -> postgres [call/shared-db]: Xoá subscription khi push provider báo 404/410 (removeSubscription) (removeById chỉ chạy khi result.removeSubscription true; adapter đặt cờ này khi statusCode 404 hoặc 410.) {apps/core-api/src/modules/notification/notification.service.ts:134, apps/core-api/src/common/platform/web-push-notification.adapter.ts:63}
17. friend -> web [return/http]: Trả kết quả tin nhắn đã gửi

## Invariants

- Tin nhắn được lưu vào PostgreSQL trước khi friend ghi streak, phát realtime và gọi notification, nên người nhận không nhận tín hiệu về tin chưa lưu; streak ghi trong try/catch riêng nên chỉ là phụ, lỗi streak không chặn tin.
- friend không tự gửi push: chỉ gọi NotificationService.create rồi sendPush, còn bảng subscription và nhà cung cấp push do notification sở hữu.
- Realtime đi hai chặng: core-api chỉ publish lên Redis, signaling-gateway mới giữ socket và relay xuống người dùng, nên friend không biết socket.
- Subscription chỉ được tạo qua POST /notifications/web-push/subscription (upsert theo endpoint); đường gửi tin nhắn chỉ đọc chúng và xoá khi push provider báo 404/410.

## Failure modes

- when Người nhận offline hoặc mất envelope pub/sub: Realtime không tới socket; thông báo đã lưu và Web Push vẫn là kênh còn lại, client đọc lại bằng REST.
- when Push provider hoặc gửi Web Push lỗi: sendPush bắt lỗi và chỉ log warn; tin nhắn và thông báo in-app đã lưu vẫn giữ. Subscription chỉ bị xoá khi provider báo 404/410, lỗi khác thì giữ lại.
- when Một trong hai người chặn người kia: isBlocked ở friend.service chạy trước khi lưu tin và ném DomainException CONVERSATION_NOT_FOUND (404), nên bước lưu, streak, phát và thông báo đều không chạy.
- when Người nhận đã mute hội thoại: friend return sớm sau khi phát realtime: không tạo thông báo in-app và không gửi push.
- when Client gửi lại cùng Idempotency-Key (replay sau timeout): ConversationService.sendMessage trả lại Message cũ nhưng FriendService vẫn chạy tiếp phần streak, realtime và tạo thông báo/push, nên người nhận có thể nhận lặp envelope và thông báo; streak thì không tăng đôi nhờ guard lastConfirmedDate.

## Uncertain

- Gateway-to-web relay dùng chung fact line 330 làm bằng chứng, không có trích dẫn riêng cho emit xuống socket.

## Endpoints covered

- POST /conversations/:id/messages [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- POST /notifications/web-push/subscription [module:notification] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/notification.md
