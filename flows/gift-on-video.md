# Tặng quà cho tác giả video

id: flow:gift-on-video
group: Tiền tệ & quà
services: module:gift, module:short-video, module:user, module:economy, module:notification

Người xem tặng quà cho tác giả video: Gift tra catalog và người nhận, Economy trừ DIA người tặng và cộng PTS tác giả trong một giao dịch, rồi Notification báo cho tác giả.

Gift điều phối, Economy ghi sổ cùng GiftEvent và thông báo trong một transaction, push gửi sau cùng.

## Participants

- web (client): Web client
- gift (service): Gift
- short-video (service): Short-video
- user (service): User
- economy (service): Economy
- notification (service): Notification
- postgres (infra): PostgreSQL

## Steps

1. web -> gift [call/http]: POST /videos/:videoId/gifts kèm giftId và Idempotency-Key (Route có Throttle 30 lần/phút và đi qua JwtAuthGuard toàn cục.)
2. gift -> short-video [call/di-call]: Lấy video, xác nhận người tặng xem được, suy ra tác giả {apps/core-api/src/modules/gift/gift.service.ts:280}
3. gift -> postgres [call/shared-db]: Đọc quà đang bán trong catalog (active = true) {apps/core-api/src/modules/gift/gift.service.ts:290}
4. gift -> user [call/di-call]: Đọc hồ sơ tác giả để biết isGuest (khách thì 0 PTS) {apps/core-api/src/modules/gift/gift.service.ts:299}
5. gift -> economy [call/di-call]: sendGift: giá DIA, PTS thưởng, idempotencyKey, callback withinTransaction {apps/core-api/src/modules/gift/gift.service.ts:308}
6. economy -> postgres [call/shared-db]: Ghi sổ kép: trừ DIA người tặng, cộng PTS tác giả, ghi outbox (LedgerService.record mở transaction và chạy callback withinTransaction bên trong; lặp lại cùng idempotencyKey thì trả bản cũ (replayed).) {apps/core-api/src/modules/economy/services/ledger.service.ts:74, apps/core-api/src/modules/economy/economy.service.ts:647}
7. gift -> postgres [call/shared-db]: Lưu GiftEvent (videoId, transactionId) cùng transaction {apps/core-api/src/modules/gift/gift.service.ts:322}
8. gift -> notification [call/di-call]: Tạo thông báo gift_received cho tác giả (cùng manager) {apps/core-api/src/modules/gift/gift.service.ts:335}
9. economy -> gift [return/di-call]: Trả transactionId và cờ replayed sau khi commit
10. gift -> postgres [call/shared-db]: Đọc lại GiftEvent theo transactionId để trả về {apps/core-api/src/modules/gift/gift.service.ts:351}
11. gift -> notification [async/di-call]: Đẩy push cho tác giả (chỉ khi không phải replay) (sendPush bọc try/catch, lỗi push chỉ ghi log warn, thông báo in-app vẫn còn.) {apps/core-api/src/modules/gift/gift.service.ts:355}
12. gift -> web [return/http]: Trả GiftEvent kèm mã quà và replayed

## Invariants

- Economy là nơi duy nhất ghi sổ và ví, nên Gift phải gọi EconomyService.sendGift thay vì tự trừ tiền; GiftEvent và thông báo chạy trong callback withinTransaction để cùng commit hoặc cùng rollback.
- Người nhận được suy từ video (authorUserId), client không gửi receiverUserId, nên không thể gửi quà lệch người; tự tặng bị chặn bằng SELF_GIFT_FORBIDDEN.
- Push chỉ gửi sau khi transaction commit và bỏ qua khi replayed, để retry cùng Idempotency-Key không trừ tiền hai lần hay báo hai lần.

## Failure modes

- when Một bước trong transaction (ghi sổ, GiftEvent hoặc thông báo) thất bại: Toàn bộ rollback: không trừ DIA, không cộng PTS, không có GiftEvent và không có thông báo.
- when Push provider lỗi hoặc ném exception: sendPush bắt lỗi và chỉ ghi log; giao dịch đã commit, thông báo in-app vẫn còn.
- when Video không nhìn thấy được, quà không active, hoặc tự tặng: Lỗi domain trả về trước khi gọi Economy, không có thay đổi dữ liệu.

## Endpoints covered

- POST /videos/:videoId/gifts [module:gift] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/gift.md
