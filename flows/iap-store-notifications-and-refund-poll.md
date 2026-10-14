# Hoàn tiền IAP: webhook Apple/Google và job quét refund

id: flow:iap-store-notifications-and-refund-poll
group: Tiền tệ & quà
services: module:economy

Khi Apple/Google hoàn tiền một giao dịch IAP, economy phải xác minh chữ ký webhook, rồi ghi bút toán đảo vào sổ cái và đánh dấu receipt đã refund trong cùng một transaction. Webhook có thể bị miss nên job quét định kỳ chủ động hỏi store để bắt lại; cả hai đường cùng dùng RefundService nên idempotent.

Webhook Apple, RTDN Google và job quét định kỳ cùng đi tới RefundService, nơi bút toán đảo và trạng thái receipt được ghi trong một transaction.

## Participants

- external:apple (external): Apple
- external:google (external): Google Play
- economy (service): Economy
- postgres (infra): PostgreSQL

## Steps

1. external:apple -> economy [call/webhook]: POST /economy/webhooks/apple kèm signedPayload (REFUND/REVOKE) (Controller là @Public (không JWT) và giới hạn 60 request/phút; xác thực bằng chữ ký của Apple.) {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:56}
2. economy -> economy [call/di-call]: Verify chữ ký signedPayload của Apple, lấy notificationType {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:62}
3. economy -> economy [call/config]: Đối chiếu bundleId; chỉ REFUND/REVOKE mới decode transactionInfo (Sai bundleId hoặc không phải refund thì trả { received: true } ngay, không ghi gì.) {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:69}
4. economy -> postgres [call/shared-db]: refundIapPurchase: tìm iap_receipt; đọc giao dịch gốc, entries, accounts (Không thấy receipt thì trả unknown_receipt; receipt đã Refunded thì trả already_refunded, không ghi thêm.) {apps/core-api/src/modules/economy/services/refund.service.ts:40, apps/core-api/src/modules/economy/services/ledger.service.ts:204}
5. economy -> postgres [call/shared-db]: ledger.reverse: bút toán đảo, ví, outbox, gốc Reversed, receipt Refunded (Giao dịch gốc không bị sửa nội dung, chỉ đổi status sang Reversed có điều kiện Completed; outbox economy.diamond.refunded được ghi cùng transaction.) {apps/core-api/src/modules/economy/services/ledger.service.ts:74, apps/core-api/src/modules/economy/services/refund.service.ts:74}
6. economy -> external:apple [return/http]: Trả 200 { received: true }
7. external:google -> economy [call/webhook]: POST /economy/webhooks/google/rtdn (Pub/Sub push) {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:93}
8. economy -> economy [call/di-call]: Verify envelope Pub/Sub qua header authorization, decode RTDN {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:100}
9. economy -> external:google [return/http]: Trả 200; CANCELED chỉ log, không tự refund (Controller chủ ý không refund ở RTDN vì CANCELED không đảm bảo nghĩa voided; nguồn refund duy nhất của Google là job quét.)
10. economy -> economy [async/config]: Job quét khởi động theo ECONOMY_REFUND_POLL_INTERVAL_MS, mỗi tick chạy runOnce (Chỉ chạy khi ECONOMY_REFUND_POLL_ENABLED bật; ManagedInterval đăng ký vào SchedulerRegistry.) {apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:66}
11. economy -> postgres [call/shared-db]: Claim lô receipt credited bằng SKIP LOCKED, stamp refund_checked_at, COMMIT ngay (Nhả lock trước khi gọi store, vì refundIapPurchase mở transaction riêng và sẽ tự deadlock nếu lock còn giữ.) {apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:92, apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:97}
12. economy -> external:apple [call/sdk]: Với từng receipt Apple: hasRefund (Get Refund History theo transaction) {apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:136}
13. external:apple -> economy [return/sdk]: Trả có/không có refund cho transaction
14. economy -> external:google [call/sdk]: findVoidedPurchaseIds(since): lấy 1 lần các purchase bị void trong window {apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:153}
15. external:google -> economy [return/sdk]: Trả tập order id đã void để đối chiếu với receipt Google
16. economy -> postgres [call/shared-db]: Receipt trùng khớp: refundIapPurchase, đọc receipt rồi ledger.reverse như bước 4-5 (Cùng RefundService nên nếu webhook đã refund trước thì kết quả là already_refunded, không hoàn lần hai.) {apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:137, apps/core-api/src/modules/economy/jobs/iap-refund-poll.service.ts:158}

## Invariants

- Bút toán đảo, việc đánh dấu giao dịch gốc Reversed và việc đổi receipt sang Refunded phải nằm trong cùng một transaction (withinTransaction của ledger.reverse), nếu không ví và receipt có thể lệch nhau.
- Webhook và job quét cùng gọi RefundService.refundIapPurchase, nên idempotent theo (provider, providerTransactionId) cộng idempotencyKey refund:provider:id; chạy trùng cũng chỉ hoàn một lần.
- Job phải COMMIT lô claim trước khi gọi API store và refundIapPurchase, vì refund mở transaction riêng trên connection khác và sẽ deadlock với lock còn giữ.
- Google RTDN CANCELED không tự refund; chỉ job quét Voided Purchases mới quyết định refund cho Google, để không đảo nhầm giao dịch hợp lệ.

## Failure modes

- when Webhook tới nhưng không tìm thấy receipt (credit chưa kịp xử lý hoặc app khác): refundIapPurchase trả unknown_receipt, webhook vẫn trả 200; job quét sẽ bắt lại nếu receipt thực sự bị lệch.
- when Webhook Apple bị miss (mạng, downtime): Job quét hỏi hasRefund theo từng receipt Apple còn credited trong window và refund bù.
- when Giao dịch gốc đã bị reverse trước đó (ví dụ admin) nhưng receipt chưa Refunded: Bắt TRANSACTION_ALREADY_REVERSED, tìm giao dịch đảo hiện có, cập nhật receipt Refunded và trả already_refunded.
- when Hai request cùng reverse một giao dịch: Update có điều kiện status Completed chỉ thắng một bên; bên thua ném TRANSACTION_ALREADY_REVERSED và rollback bút toán đảo của mình.
- when Lô claim đầy batchSize (200): Job ghi cảnh báo; refund_checked_at xoay vòng để các tick sau quét sang receipt khác.

## Uncertain

- Verifier thật hay bản Dev (không verify chữ ký) được dùng phụ thuộc cấu hình môi trường lúc chạy; code có lớp Dev chặn ở production nhưng việc binding thực tế không quyết định được chỉ từ mã nguồn trace này.
- Kiểm tra bundleId Apple chỉ chạy khi ECONOMY_APPLE_BUNDLE_ID khác rỗng, giá trị thực tế là cấu hình triển khai.

## Endpoints covered

- POST /economy/webhooks/apple [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- POST /economy/webhooks/google/rtdn [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- IapRefundPollService.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
