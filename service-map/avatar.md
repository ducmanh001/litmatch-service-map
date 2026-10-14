# Service map: Avatar

id: service-map:avatar
module: module:avatar
deployable: core-api
confidence: high

Catalog, sở hữu và trang bị avatar

Sở hữu catalog layer avatar (avatar_assets, giá là DATA), quyền sở hữu item (user_avatar_items) và cấu hình đang trang bị (user_avatar_configs, 5 slot). Mua item trả phí là việc của Economy (Avatar chỉ gọi spendDiamond), Avatar không ghi ví/sổ cái và không đụng User.avatarId.

Owns: bảng avatar_assets (slot, code, imageUrl, zIndex, priceDiamond, active); bảng user_avatar_items (unique userId+avatarAssetId = nguồn sự thật 'đã sở hữu'); bảng user_avatar_configs (1 dòng/user, 5 cột asset id đang trang bị, lazy-init)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/avatar/catalog | jwt | Liệt kê catalog item đang bật, lọc theo slot tuỳ chọn, sắp theo sortOrder/code; giá do server quyết định. Cần JWT (JwtAuthGuard toàn cục, không @Public). |
| e2 | rest | GET | /api/v1/avatar/me/items | jwt | Liệt kê các item avatar user hiện tại đã sở hữu (join user_avatar_items với avatar_assets). |
| e3 | rest | POST | /api/v1/avatar/items/:assetId/claim | jwt | Nhận item miễn phí (priceDiamond = 0): ghi sở hữu idempotent ON CONFLICT DO NOTHING, không đi qua Economy; item có giá bị 400 ASSET_REQUIRES_PURCHASE. Trả 204. |
| e4 | rest | POST | /api/v1/avatar/items/:assetId/buy | jwt | Mua item trả phí: bắt buộc Idempotency-Key, Economy.spendDiamond trừ DIA (TransactionType.AvatarPurchase) rồi mới ghi sở hữu idempotent; item miễn phí bị 400 ASSET_IS_FREE; replay không trừ tiền lần hai. |
| e5 | rest | PUT | /api/v1/avatar/me/equip | jwt | Trang bị item vào slot: chống IDOR bằng 3 check (asset active, đúng slot, user đã sở hữu) rồi cập nhật cột slot trên user_avatar_configs, lazy-init config nếu chưa có; trả layer đã resolve. |
| e6 | rest | GET | /api/v1/avatar/me | jwt | Avatar của chính mình: config + danh sách layer đầy đủ để client ghép theo zIndex; lazy-init config mặc định ở lần gọi đầu. |
| e7 | rest | GET | /api/v1/avatar/users/:userId | jwt | Xem avatar người khác: chỉ đọc, không cần là bạn bè (avatar là thông tin hiển thị công khai), chưa có config thì trả layers rỗng; vẫn cần JWT. |

## Coordination edges (why this service talks to others)

### trừ DIA mua item (c1) with economy [service, out, di-call]

- endpoints: e4
- what: EconomyService.spendDiamond(userId, TransactionType.AvatarPurchase, asset.priceDiamond, avatar:buy idempotency key, {avatarAssetId, assetCode}) ghi ledger Nợ user_wallet / Có system_revenue và trả {transactionId, replayed}.
- why: Ví và LedgerEntry chỉ Economy được ghi; check đủ số dư và khoá ví FOR UPDATE nằm trong ledger.record. Idempotency key unique ở Transaction là thứ cho phép retry an toàn: vì spendDiamond và ghi sở hữu là hai bước tách rời, replay cùng key không trừ lần 2 mà vẫn cho Avatar hoàn tất grantOwnership.
- if missing: Avatar phải tự trừ ví (vi phạm 'Economy là nguồn sự thật'), hoặc item được cấp không trả tiền; retry sau crash sẽ trừ DIA hai lần.
- evidence: apps/core-api/src/modules/avatar/avatar.service.ts:85
- review: sound — applyWalletDeltas kiểm số dư tại thời điểm trừ sau lockWallet FOR UPDATE; buy gọi spendDiamond rồi grantOwnership (avatar.service:85-92).

### catalog / sở hữu / config (c2) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7
- what: Đọc avatar_assets (active, theo slot; giá đọc lại lúc mua), ghi user_avatar_items qua INSERT ... ON CONFLICT DO NOTHING và cập nhật user_avatar_configs theo slot.
- why: Unique (userId, avatarAssetId) là nguồn sự thật 'đã sở hữu' và là thứ làm grantOwnership idempotent sau spendDiamond; equip phải đọc bảng này để chặn trang bị item của người khác/chưa mua (IDOR). Giá nằm trong DB nên server không tin giá client.
- if missing: Không chứng minh được quyền sở hữu khi equip (ai cũng đeo được item trả phí), và retry buy có thể tạo bản ghi sở hữu trùng.
- evidence: apps/core-api/src/modules/avatar/entities/avatar-asset.entity.ts:16, apps/core-api/src/modules/avatar/entities/user-avatar-config.entity.ts:8, apps/core-api/src/modules/avatar/entities/user-avatar-item.entity.ts:13
- review: sound — Index unique uq_user_avatar_items_pair (userId, avatarAssetId) và grantOwnership dùng orIgnore; equip kiểm owned bằng itemRepo.exists.

### Idempotency-Key, equip (c50) with web [client, both, http]

- endpoints: e3, e4, e5, e6
- what: Web gọi POST /avatar/items/{assetId}/buy với header Idempotency-Key sinh theo intent (giữ qua retry), claim item free rồi PUT /avatar/me/equip, sau đó ghép các layer theo zIndex server đã sắp.
- why: Buy và grant sở hữu không cùng transaction nên Idempotency-Key ổn định của client là cơ chế duy nhất để retry sau lỗi mạng/crash mà không trừ DIA hai lần; mua và trang bị là hai call riêng nên client phải gọi equip sau khi buy/claim thành công.
- if missing: Retry mạng trừ DIA lần nữa (hoặc bị từ chối do thiếu header), và item đã mua không hiện trên avatar vì không có bước equip.
- evidence: apps/web/src/features/avatar/api.ts:73, apps/web/src/features/avatar/components/avatar-builder.tsx:64
- review: sound — Web gửi Idempotency-Key cho buy, claim rồi equip tách call; buy và grant không cùng transaction đúng như avatar.service.

### imageUrl trỏ static /avatar (c51) with web [client, in, config]

- endpoints: e1, e6, e7
- what: avatar_assets.image_url được seed bằng path tương đối '/avatar/<code>.svg' trỏ tới file tĩnh apps/web/public/avatar; client render mỗi layer bằng <img src={layer.imageUrl}> với zIndex từ server.
- why: Suy luận: Avatar chỉ lưu URL layer chứ không phục vụ file, nên asset phải được deploy cùng web đúng path trùng với seed; thêm item mới cần thêm cả dòng DB lẫn file SVG tương ứng ở web.
- if missing: Dòng catalog tồn tại nhưng ảnh 404: layer đã mua/trang bị hiển thị trống trên avatar.
- evidence: apps/core-api/src/database/migrations/1757300000000-avatar-public-assets.ts:6, apps/web/src/features/avatar/components/avatar-builder.tsx:47
- review: sound — Migration seed 8 path /avatar/*.svg và apps/web/public/avatar có đúng 8 file SVG tương ứng.

## Notes

- Khác Gift: buy() KHÔNG atomic với ledger. EconomyService.spendDiamond tự mở transaction riêng rồi AvatarService.grantOwnership (ON CONFLICT DO NOTHING) chạy SAU, ngoài transaction đó (avatar.service.ts:85-92). Nếu crash giữa hai bước thì tiền đã trừ mà chưa có item cho đến khi client retry cùng Idempotency-Key (replay không trừ lần 2, rồi ghi sở hữu). Doc avatar-service.md § 2 ghi đúng thiết kế này; không có job đối soát nào tự sửa trường hợp client không retry.
- Economy có spendDiamondInTransaction(manager, ...) cho phép cùng transaction (Matching dùng), nhưng Avatar vẫn dùng spendDiamond generic; hệ quả là cửa sổ lệch nói trên là chủ ý theo doc, không phải thiếu sót của doc.
- AvatarModule exports: [] và dossier không thấy module nào gọi vào Avatar; admin cũng không có route quản lý avatar_assets (catalog chỉ đổi bằng migration/seed), khác Gift có AdminModule.
- Không có realtime/job/event nào ở Avatar: người khác thấy avatar mới chỉ khi tự GET /avatar/users/:userId.
- imageUrl trong seed là path tương đối '/avatar/*.svg' (migration 1757300000000) trỏ vào static của apps/web/public/avatar, không phải object storage/CDN.
- Đã xác minh grantOwnership chạy vô điều kiện sau spendDiamond kể cả khi replayed=true (avatar.service.ts:85-92) và unique uq_user_avatar_items_pair (userId, avatarAssetId) làm nó idempotent, nên retry cùng Idempotency-Key luôn hoàn tất việc cấp item. Ngược lại với Matching speed-up, nơi boost bị bỏ qua khi replayed=true (xem economy).
