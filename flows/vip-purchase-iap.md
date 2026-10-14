# Mua VIP và xác minh biên nhận IAP

id: flow:vip-purchase-iap
group: Tiền tệ & quà
services: module:economy

Hai luồng tiền chạy qua cùng một sổ cái: mua VIP bằng diamond với Idempotency-Key, và nạp diamond qua IAP chỉ khi server tự hỏi Apple/Google. Economy phải ghi sổ, ví, VIP/biên nhận trong một giao dịch PostgreSQL để không trừ/cộng lệch.

Economy gom ghi sổ + ví + VIP (hoặc biên nhận IAP) vào một transaction PostgreSQL; biên nhận IAP được xác minh với store trước khi ghi sổ.

## Participants

- web (client): Web client
- economy (service): Economy
- postgres (infra): PostgreSQL
- external:app-store-google-play (external): Apple / Google Play

## Steps

1. web -> economy [call/http]: POST /economy/vip/purchase kèm Idempotency-Key và planId
2. economy -> postgres [call/shared-db]: Đọc gói VIP đang active (VipPlan) theo planId {apps/core-api/src/modules/economy/economy.service.ts:377}
3. economy -> postgres [call/shared-db]: LedgerService.record: tra khóa idempotency vip:{userId}:{key}, có rồi thì trả replay {apps/core-api/src/modules/economy/services/ledger.service.ts:61}
4. economy -> postgres [call/shared-db]: Mở transaction: ghi giao dịch, khóa ví FOR UPDATE, ghi ledger_entries, cập nhật số dư (Kiểm tra đủ số dư nằm trong lúc khóa ví (WALLET_INSUFFICIENT_BALANCE), không ở đầu luồng.) {apps/core-api/src/modules/economy/services/ledger.service.ts:74, apps/core-api/src/modules/economy/services/ledger.service.ts:420}
5. economy -> postgres [call/shared-db]: Cùng transaction: gia hạn VIP trên ví, ghi OutboxEvent economy.vip.purchased {apps/core-api/src/modules/economy/economy.service.ts:423, apps/core-api/src/modules/economy/economy.service.ts:428}
6. economy -> web [return/http]: Trả transactionId, tier, vipExpiresAt, replayed
7. web -> economy [call/http]: POST /economy/iap/verify kèm provider, productId, payload biên nhận
8. economy -> postgres [call/shared-db]: Đọc IapProduct active theo productId + provider để biết số diamond {apps/core-api/src/modules/economy/economy.service.ts:300}
9. economy -> external:app-store-google-play [call/http]: IapVerifier.verify: server tự xác minh biên nhận với store, không tin client (Chỉ implementation 'store' gọi Apple/Google; 'dev' chấp nhận devTransactionId giả, 'disabled' từ chối.) {apps/core-api/src/modules/economy/economy.service.ts:313, apps/core-api/src/modules/economy/clients/apple-receipt-api.adapter.ts:100}
10. external:app-store-google-play -> economy [return/http]: Trả providerTransactionId đã xác minh
11. economy -> postgres [call/shared-db]: ledger.record, khóa iap:{provider}:{providerTransactionId}: ghi sổ và cộng ví {apps/core-api/src/modules/economy/economy.service.ts:322, apps/core-api/src/modules/economy/services/ledger.service.ts:74}
12. economy -> postgres [call/shared-db]: Cùng transaction: lưu IapReceipt gắn transactionId của bút toán {apps/core-api/src/modules/economy/economy.service.ts:346}
13. economy -> web [return/http]: Trả transactionId, diamonds, replayed

## Invariants

- Gia hạn VIP và OutboxEvent được ghi trong callback withinTransaction của LedgerService, cùng transaction với ledger_entries và ví khóa FOR UPDATE: trừ diamond và cấp VIP thành công hoặc hỏng cùng nhau.
- Khóa idempotency được tra trước khi mở transaction và còn được kiểm lại trong transaction, nên client gửi lại cùng Idempotency-Key chỉ nhận replay, không gia hạn lần hai.
- IAP: verify với store phải xong trước ledger.record; khóa idempotency lấy từ providerTransactionId do server xác minh, nên gửi lại receipt không cộng diamond đôi.
- IapReceipt được lưu trong cùng transaction với bút toán và mang transactionId, để không có biên nhận mồ côi hoặc bút toán không có biên nhận.

## Failure modes

- when Gói VIP không tồn tại hoặc không active: Ném VIP_PLAN_UNKNOWN (404) trước khi chạm sổ cái.
- when Số dư ví không đủ lúc trừ diamond mua VIP: Ledger ném WALLET_INSUFFICIENT_BALANCE; transaction rollback, không cấp VIP.
- when productId IAP không có trong catalog: Ném IAP_PRODUCT_UNKNOWN (400), không gọi store và không ghi sổ.
- when Verifier ở chế độ 'disabled' hoặc receipt không hợp lệ: Verifier ném lỗi (IAP_DISABLED 403 / IAP_RECEIPT_INVALID 400) trước mọi ghi sổ.
- when Hai request cùng khóa idempotency chạy đồng thời: Insert có orIgnore; request thua đọc lại bản thắng và trả replayed=true nếu cùng request hash.

## Endpoints covered

- POST /economy/vip/purchase [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- POST /economy/iap/verify [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
