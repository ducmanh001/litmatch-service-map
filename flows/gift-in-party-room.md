# Tặng quà trong phòng party

id: flow:gift-in-party-room
group: Tiền tệ & quà
services: module:gift, module:party-room, module:economy, module:notification, module:signaling-gateway

Người dùng tặng quà trong phòng party đang diễn ra. Gift giữ catalog và GiftEvent, Economy là nơi duy nhất ghi sổ cái, Notification ghi thông báo, rồi cả phòng nhận hiệu ứng realtime qua Redis và signaling-gateway.

Luồng tặng quà từ request của web, qua giao dịch tiền và GiftEvent, tới realtime cho cả phòng.

## Participants

- web (client): Web client
- gift (service): Gift
- party-room (service): Party Room
- economy (service): Economy
- notification (service): Notification
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> gift [call/http]: POST /party/rooms/:roomId/gifts kèm giftId và idempotencyKey (Mọi route REST đều qua JwtAuthGuard toàn cục.) {apps/core-api/src/app/app.module.ts:154}
2. gift -> postgres [call/shared-db]: Đọc quà đang bán, giá lấy từ DB lúc tặng {apps/core-api/src/modules/gift/gift.service.ts:172}
3. gift -> party-room [call/di-call]: Lấy member active, kiểm tra người tặng và người nhận đều ở trong phòng (Đây là kiểm tra nghiệp vụ, không phải chốt an toàn tiền. Gift cũng đọc user nhận để biết guest (không có điểm).) {apps/core-api/src/modules/gift/gift.service.ts:184}
4. gift -> economy [call/di-call]: Gọi sendGift: giá, điểm PTS, idempotencyKey và callback trong giao dịch {apps/core-api/src/modules/gift/gift.service.ts:210}
5. economy -> postgres [call/shared-db]: Ghi bút toán sổ cái và sự kiện outbox economy.gift.sent (Giao dịch DB mở ở LedgerService.record (dataSource.transaction), callback withinTransaction chạy trong cùng manager.) {apps/core-api/src/modules/economy/economy.service.ts:647}
6. gift -> postgres [call/shared-db]: Cùng giao dịch: lưu GiftEvent gắn transactionId {apps/core-api/src/modules/gift/gift.service.ts:225}
7. gift -> notification [call/di-call]: Cùng giao dịch: tạo thông báo GiftReceived cho người nhận {apps/core-api/src/modules/gift/gift.service.ts:238}
8. economy -> gift [return/di-call]: Trả transactionId và cờ replayed {apps/core-api/src/modules/economy/economy.service.ts:665}
9. gift -> postgres [call/shared-db]: Đọc lại GiftEvent theo transactionId {apps/core-api/src/modules/gift/gift.service.ts:255}
10. gift -> redis [async/redis-pubsub]: Publish sự kiện quà tới từng member active của phòng (Chỉ chạy khi không phải replay.) {apps/core-api/src/modules/gift/gift.service.ts:498, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
11. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
12. signaling-gateway -> web [async/socket]: Relay hiệu ứng quà xuống socket của từng member {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
13. gift -> notification [async/di-call]: Đẩy push cho người nhận (best-effort) {apps/core-api/src/modules/gift/gift.service.ts:261}
14. gift -> web [return/http]: Trả giftEvent, gift và replayed

## Invariants

- Economy là nơi duy nhất ghi sổ cái, nên Gift phải gọi economy.sendGift thay vì tự trừ tiền.
- GiftEvent và thông báo được ghi qua callback withinTransaction cùng manager với bút toán, nên không có quà mà thiếu tiền hoặc ngược lại.
- Realtime và push chỉ chạy sau khi giao dịch xong và không phải replay, nên retry cùng idempotencyKey không bắn hiệu ứng lần hai.

## Failure modes

- when Redis publish, fanout hoặc push thất bại sau khi giao dịch đã commit: Lỗi bị nuốt và chỉ log warn: publishRealtimeEvent có try/catch, publishGiftSent bọc try/catch, sendPush bọc try/catch. Tiền và GiftEvent vẫn đúng, request vẫn trả 200, member có thể không nhận hiệu ứng realtime.
- when Người nhận rời phòng đúng lúc đang tặng: Tiền vẫn đúng nhờ giao dịch và idempotency ở economy, theo comment trong gift.service.ts.
- when Client gửi lại cùng idempotencyKey: Economy trả replayed, Gift trả lại GiftEvent cũ và bỏ qua realtime và push.

## Endpoints covered

- POST /party/rooms/:roomId/gifts [module:gift] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/gift.md
