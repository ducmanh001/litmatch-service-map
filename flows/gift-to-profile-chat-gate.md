# Tặng quà từ hồ sơ để mở chat (cổng quà)

id: flow:gift-to-profile-chat-gate
group: Tiền tệ & quà
services: module:gift, module:friend, module:user, module:economy, module:notification

Người dùng tặng quà từ profile người khác để mở chat 1-1. Gift tính điểm, Economy ghi sổ kép, Friend tạo conversation và liên hệ first-contact, Notification ghi thông báo; tất cả phải nằm trong cùng một giao dịch DB để không có quà mà thiếu chat hoặc ngược lại.

Gift điều phối: kiểm tra chặn, Economy mở giao dịch ledger, rồi ghi GiftEvent + conversation + thông báo trong cùng transaction, push sau commit.

## Participants

- web (client): Web client
- gift (service): Gift
- friend (service): Friend
- user (service): User
- economy (service): Economy
- notification (service): Notification
- postgres (infra): PostgreSQL

## Steps

1. web -> gift [call/http]: POST /profiles/:profileUserId/gifts kèm giftId và idempotency key (Route có JWT guard toàn cục và @Throttle 30 request/phút.)
2. gift -> friend [call/di-call]: Kiểm tra hai bên không chặn nhau (assertNotBlocked) {apps/core-api/src/modules/gift/gift.service.ts:382}
3. gift -> postgres [call/shared-db]: Đọc quà đang bán (active) trong catalog gifts {apps/core-api/src/modules/gift/gift.service.ts:384}
4. gift -> user [call/di-call]: Lấy người nhận để biết isGuest và tính điểm PTS {apps/core-api/src/modules/gift/gift.service.ts:392}
5. gift -> economy [call/di-call]: sendGift: giá, điểm, idempotencyKey và callback withinTransaction {apps/core-api/src/modules/gift/gift.service.ts:401}
6. economy -> postgres [call/shared-db]: Mở transaction: ghi sổ kép trừ diamond người tặng + outbox economy.gift.sent {apps/core-api/src/modules/economy/economy.service.ts:638, apps/core-api/src/modules/economy/services/ledger.service.ts:74}
7. economy -> gift [call/di-call]: Gọi lại callback withinTransaction với manager và transactionId {apps/core-api/src/modules/economy/economy.service.ts:662}
8. gift -> friend [call/di-call]: Kiểm tra chặn lần nữa và khóa profile để giữ ngưỡng first-contact {apps/core-api/src/modules/gift/gift.service.ts:420}
9. gift -> postgres [call/shared-db]: Lưu GiftEvent (gift_events) gắn transactionId {apps/core-api/src/modules/gift/gift.service.ts:421}
10. gift -> friend [call/di-call]: ensureConversationInManager: tạo/lấy conversation 1-1 {apps/core-api/src/modules/gift/gift.service.ts:435}
11. gift -> friend [call/di-call]: Nếu conversation mới: ghi ProfileChatContact (first-contact) {apps/core-api/src/modules/gift/gift.service.ts:441}
12. gift -> notification [call/di-call]: createWithManager: ghi thông báo GiftReceived cho người nhận {apps/core-api/src/modules/gift/gift.service.ts:447}
13. gift -> postgres [call/shared-db]: Sau commit: đọc lại GiftEvent theo transactionId để trả về {apps/core-api/src/modules/gift/gift.service.ts:463}
14. gift -> notification [async/di-call]: sendPush chỉ khi không phải replay (sendPush bọc try/catch, lỗi push chỉ log warn và không throw ra caller.) {apps/core-api/src/modules/gift/gift.service.ts:467}
15. gift -> web [return/http]: Trả GiftEventDto kèm cờ replayed

## Invariants

- Ledger, GiftEvent, conversation, ProfileChatContact và thông báo nằm cùng một transaction do Economy mở, nên lỗi ở bước nào cũng rollback toàn bộ: không mất diamond mà thiếu chat.
- Khóa profile (lockProfileForChatInManager) chạy trong transaction để hai request đồng thời không cùng vượt ngưỡng first-contact.
- assertNotBlocked được kiểm tra lại trong transaction để block phát sinh sau precheck không biến quà thành đường bypass Safety.
- Push chỉ gửi sau commit và chỉ khi không phải replay, nên retry cùng idempotency key không gửi push lần hai.

## Failure modes

- when Một bước trong callback withinTransaction (ví dụ tạo conversation) ném lỗi: Cả transaction rollback, ledger và GiftEvent không được ghi.
- when Push provider hoặc web push lỗi: sendPush chỉ log cảnh báo; thông báo in-app vẫn còn và request vẫn thành công.
- when Hai bên có chặn nhau hoặc quà không active: assertNotBlocked hoặc kiểm tra quà ném lỗi trước khi Economy mở transaction.

## Endpoints covered

- POST /profiles/:profileUserId/gifts [module:gift] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/gift.md
