# Trả lời story thành tin nhắn

id: flow:story-reply-to-dm
group: Xã hội & nội dung
services: module:feed, module:safety, module:user, module:friend, module:signaling-gateway

Người xem trả lời một story: Feed không có chat riêng nên phải hỏi Safety/Friend/User xem có được xem story không, rồi nhờ Friend tạo DM thật kèm attachment snapshot mediaUrl, vì story hết hạn còn message thì sống mãi.

Feed kiểm tra quyền xem story rồi giao việc gửi DM cho Friend, Friend lưu message và đẩy realtime tới hai bên qua Redis và gateway.

## Participants

- web (client): Web client
- feed (service): Feed
- safety (service): Safety
- user (service): User
- friend (service): Friend
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> feed [call/http]: POST /stories/:storyId/reply kèm nội dung và Idempotency-Key (Mọi route REST đều qua JwtAuthGuard toàn cục trừ @Public.)
2. feed -> postgres [call/shared-db]: Đọc Story, kiểm tra tồn tại và chưa hết hạn {apps/core-api/src/modules/feed/services/story.service.ts:119}
3. feed -> safety [call/di-call]: isBlocked hai chiều giữa người xem và tác giả story {apps/core-api/src/modules/feed/services/story.service.ts:129}
4. feed -> friend [call/di-call]: areFriends khi story không public (audience) {apps/core-api/src/modules/feed/services/story.service.ts:133}
5. feed -> user [call/di-call]: isHidden: tác giả có đang ẩn hồ sơ không (Vi phạm block, hidden hoặc audience đều trả cùng lỗi STORY_NOT_FOUND (oracle-safe).) {apps/core-api/src/modules/feed/services/story.service.ts:134}
6. feed -> friend [call/di-call]: getConversationWithFriend: cần Conversation thật với tác giả (Lỗi bất kỳ được dịch thành STORY_REPLY_REQUIRES_FRIENDSHIP (403), ví dụ story public của người lạ.) {apps/core-api/src/modules/feed/services/story.service.ts:203}
7. feed -> friend [call/di-call]: sendMessage kèm attachment story_reply snapshot mediaUrl {apps/core-api/src/modules/feed/services/story.service.ts:212, apps/core-api/src/modules/feed/services/story.service.ts:217}
8. friend -> safety [call/di-call]: isBlocked hai chiều giữa người gửi và đối tác chat {apps/core-api/src/modules/friend/friend.service.ts:346}
9. friend -> postgres [call/shared-db]: Lưu Message với idempotencyKey, attachment; cập nhật lastMessageAt (Message được save trước; lastMessageAt cập nhật bằng câu UPDATE riêng (không cùng transaction, chỉ để sort danh sách). Trùng Idempotency-Key cùng nội dung thì trả message cũ và bỏ qua UPDATE; khác nội dung thì lỗi 409.) {apps/core-api/src/modules/friend/services/conversation.service.ts:162}
10. friend -> redis [async/redis-pubsub]: Publish friend.message tới kênh realtime của cả hai user {apps/core-api/src/modules/friend/friend.service.ts:425, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
11. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
12. signaling-gateway -> web [async/socket]: Emit friend.message tới socket room của người dùng {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
13. feed -> web [return/http]: Trả Message vừa tạo (201) cho người trả lời

## Invariants

- Feed không sở hữu Conversation/Message nên phải đi qua FriendService.sendMessage để tái dùng idempotency, block hai chiều, realtime và notification của chat.
- Attachment snapshot mediaUrl được copy vào message vì story hết hạn sau TTL còn message sống mãi; attachment chỉ gắn được qua DI nội bộ, HTTP không tự gắn kind tuỳ ý.
- Publish realtime chạy sau khi Message đã lưu (sau cả bước tính streak) và lỗi publish chỉ bị log warn nên không làm hỏng việc gửi; khi replay Idempotency-Key, friend.service vẫn publish lại envelope.

## Failure modes

- when Story hết hạn, bị block, tác giả ẩn hồ sơ hoặc sai audience: Trả STORY_NOT_FOUND (404) cùng mã lỗi cho mọi trường hợp, không lộ lý do.
- when Chưa có Conversation với tác giả (ví dụ story public của người lạ): Feed dịch thành STORY_REPLY_REQUIRES_FRIENDSHIP (403).
- when Publish Redis thất bại: publishRealtimeEvent bắt lỗi và log warn; message đã lưu, client còn polling fallback.
- when Lỗi tạo notification hoặc tính streak sau khi lưu message: friend.service bắt lỗi và log, message vẫn gửi thành công.

## Endpoints covered

- POST /stories/:storyId/reply [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
