# Mua vật phẩm avatar bằng kim cương

id: flow:avatar-buy-item
group: Tiền tệ & quà
services: module:avatar, module:economy

Người dùng nhận item miễn phí, mua item trả phí bằng DIA với Idempotency-Key rồi trang bị. Avatar sở hữu catalog và quyền sở hữu nhưng không được tự trừ tiền, nên phải gọi Economy (sổ cái duy nhất) trước khi ghi quyền sở hữu.

Ba endpoint claim, buy, equip của Avatar và chỗ Avatar gọi Economy.spendDiamond trước khi ghi sở hữu item.

## Participants

- web (client): Web client
- avatar (service): Avatar
- economy (service): Economy
- postgres (infra): PostgreSQL

## Steps

1. web -> avatar [call/http]: POST /avatar/items/:assetId/claim (nhận item miễn phí) (Mọi route REST đều qua JwtAuthGuard.)
2. avatar -> postgres [call/shared-db]: Đọc asset active, kiểm giá = 0, ghi user_avatar_items (bỏ qua nếu đã có) {apps/core-api/src/modules/avatar/avatar.service.ts:160, apps/core-api/src/modules/avatar/avatar.service.ts:176}
3. avatar -> web [return/http]: 204 No Content, claim idempotent
4. web -> avatar [call/http]: POST /avatar/items/:assetId/buy kèm header Idempotency-Key
5. avatar -> postgres [call/shared-db]: Đọc asset active và giá priceDiamond (giá = 0 thì báo dùng claim) {apps/core-api/src/modules/avatar/avatar.service.ts:160}
6. avatar -> economy [call/di-call]: spendDiamond(AvatarPurchase, giá, key avatarBuyIdempotencyKey) {apps/core-api/src/modules/avatar/avatar.service.ts:85}
7. economy -> postgres [call/shared-db]: ledger.record: trừ ví user, cộng SystemRevenue, replay nếu key đã có (record() tra idempotencyKey trước; nếu chưa có thì mở transaction recordInManager; nếu thua unique key thì đọc lại giao dịch thắng.) {apps/core-api/src/modules/economy/services/ledger.service.ts:74}
8. economy -> avatar [return/di-call]: Trả { transactionId, replayed }
9. avatar -> postgres [call/shared-db]: Ghi user_avatar_items idempotent (ON CONFLICT DO NOTHING) {apps/core-api/src/modules/avatar/avatar.service.ts:181}
10. avatar -> web [return/http]: 200 { replayed }
11. web -> avatar [call/http]: PUT /avatar/me/equip { slot, avatarAssetId }
12. avatar -> postgres [call/shared-db]: Đọc asset, kiểm đúng slot và đã sở hữu (exists user_avatar_items) {apps/core-api/src/modules/avatar/avatar.service.ts:110}
13. avatar -> postgres [call/shared-db]: Nếu chưa có config: tạo mặc định (cấp item free + ghi config) trong transaction {apps/core-api/src/modules/avatar/avatar.service.ts:234}
14. avatar -> postgres [call/shared-db]: Cập nhật cột slot trong user_avatar_configs {apps/core-api/src/modules/avatar/avatar.service.ts:123}
15. avatar -> postgres [call/shared-db]: getMyAvatar: đọc config và các layer avatar_assets theo zIndex {apps/core-api/src/modules/avatar/avatar.service.ts:214}
16. avatar -> web [return/http]: Trả config và layers đang trang bị

## Invariants

- Economy.spendDiamond không có hook withinTransaction, nên Avatar bắt buộc trừ tiền trước rồi mới ghi sở hữu; cả hai nhánh đều idempotent nên retry cùng Idempotency-Key an toàn.
- Chỉ Economy ghi ledger/wallet; Avatar chỉ gọi spendDiamond và không tự trừ diamond, giá lấy từ asset.priceDiamond trong catalog.
- equip kiểm item đã sở hữu và đúng slot trước khi cập nhật config, tránh trang bị item của người khác (IDOR).

## Failure modes

- when Trừ tiền thành công nhưng ghi user_avatar_items lỗi: Item chưa được cấp; client retry cùng Idempotency-Key sẽ replay phần trừ tiền và ghi lại sở hữu, không trừ hai lần.
- when Gọi buy cho item miễn phí hoặc claim cho item trả phí: Trả 400 (ASSET_IS_FREE / ASSET_REQUIRES_PURCHASE), không đụng Economy.
- when equip item chưa sở hữu: Trả 403 ITEM_NOT_OWNED, config không đổi.

## Endpoints covered

- POST /avatar/items/:assetId/claim [module:avatar] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/avatar.md
- POST /avatar/items/:assetId/buy [module:avatar] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/avatar.md
- PUT /avatar/me/equip [module:avatar] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/avatar.md
