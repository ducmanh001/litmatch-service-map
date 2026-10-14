# Nạp kim cương qua PayOS và webhook

id: flow:payos-topup-webhook
group: Tiền tệ & quà
services: module:economy, module:external:payos

Người dùng tạo đơn nạp PayOS rồi thanh toán trên trang của PayOS; tiền chỉ được cộng vào ví khi webhook có chữ ký hợp lệ quay lại economy, ghi sổ cái kép và chống cộng hai lần. Economy phải phối hợp với PayOS (bên ngoài) và PostgreSQL.

Tạo đơn và link thanh toán PayOS, sau đó webhook đã xác thực được đối chiếu, ghi sổ cái kép và đánh dấu đơn Paid; client poll trạng thái đơn để biết khi nào Paid.

## Participants

- web (client): Web client
- economy (service): Economy
- postgres (infra): PostgreSQL
- external:payos (service): PayOS

## Steps

1. web -> economy [call/http]: POST /economy/payos/orders với gói nạp và Idempotency-Key
2. economy -> postgres [call/shared-db]: Tra đơn theo idempotencyKey, nếu chưa có thì lưu đơn từ gói đang bán {apps/core-api/src/modules/economy/services/payos.service.ts:87, apps/core-api/src/modules/economy/services/payos.service.ts:92}
3. economy -> external:payos [call/http]: Tạo payment link cho orderCode, số tiền VND và URL quay lại (Lời gọi nằm ngoài khóa hàng và connection theo chú thích trong mã.) {apps/core-api/src/modules/economy/services/payos.service.ts:239}
4. economy -> postgres [call/shared-db]: Lưu link thanh toán vào đơn, chỉ ghi khi paymentLinkId còn trống {apps/core-api/src/modules/economy/services/payos.service.ts:250}
5. economy -> web [return/http]: Trả thông tin đơn kèm checkoutUrl và qrCode
6. external:payos -> economy [call/webhook]: POST /economy/webhooks/payos kèm chữ ký, HMAC được kiểm trước mọi truy vấn {apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:121, apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:125}
7. economy -> postgres [call/shared-db]: Tìm đơn theo orderCode và đối chiếu trạng thái, số tiền, tiền tệ, paymentLinkId {apps/core-api/src/modules/economy/services/payos.service.ts:171}
8. economy -> postgres [call/shared-db]: Mở giao dịch DB, ghi sổ cái kép payos-credit (Debit SystemIap, Credit UserWallet) (Giao dịch mở ở LedgerService.record bằng dataSource.transaction; callback withinTransaction chạy cuối cùng trong cùng manager.) {apps/core-api/src/modules/economy/services/payos.service.ts:178, apps/core-api/src/modules/economy/services/payos.service.ts:176, apps/core-api/src/modules/economy/services/ledger.service.ts:74}
9. economy -> postgres [call/shared-db]: Trong withinTransaction: khóa đơn pessimistic_write, lệch snapshot thì rollback (Khóa đơn nằm SAU khi sổ cái và ví đã ghi trong cùng giao dịch, không phải trước.) {apps/core-api/src/modules/economy/services/payos.service.ts:316}
10. economy -> postgres [call/shared-db]: Đánh dấu đơn Paid và gắn transactionId của bút toán {apps/core-api/src/modules/economy/services/payos.service.ts:208}
11. economy -> external:payos [return/webhook]: Trả 200 {received: true} cho webhook
12. web -> economy [call/http]: GET /economy/payos/orders/:orderId poll trạng thái đơn (Client hỏi lại sau khi thanh toán để chờ webhook payOS; trạng thái Paid chỉ xuất hiện sau khi webhook đã được xác thực và ghi sổ (bước 6-10).) {apps/core-api/src/modules/economy/economy.controller.ts:98}
13. economy -> postgres [call/shared-db]: Đọc đơn theo id và userId, đơn Pending quá hạn thì chuyển Expired (Chỉ đọc đơn của chính người gọi; không thấy đơn thì ném lỗi không tìm thấy đơn. Ghi duy nhất của đường này là UPDATE có điều kiện status Pending sang Expired khi expiresAt đã qua, rồi đọc lại đơn.) {apps/core-api/src/modules/economy/services/payos.service.ts:145, apps/core-api/src/modules/economy/services/payos.service.ts:151}
14. economy -> web [return/http]: Trả status, transactionId và số kim cương của đơn (transactionId là null cho tới khi webhook gắn bút toán vào đơn (bước 10).)

## Invariants

- Webhook là route public nên HMAC phải được kiểm trước mọi truy vấn hay ghi, nếu không ai cũng có thể giả lệnh cộng tiền.
- Trong cùng giao dịch với sổ cái, đơn được khóa FOR UPDATE và đối chiếu lại số tiền, tiền tệ, paymentLinkId; nếu lệch thì ném lỗi và rollback toàn bộ, nên webhook đến trễ hay trùng không cộng sai.
- Sổ cái dùng idempotencyKey payos-credit theo orderCode, nên webhook gửi lại chỉ replay chứ không cộng ví hai lần.
- Đường poll GET /economy/payos/orders/:orderId chỉ đọc: nó không bao giờ tạo bút toán, chỉ có webhook đã xác thực mới ghi sổ và đánh dấu Paid; ghi duy nhất của poll là chuyển đơn Pending quá hạn sang Expired.

## Failure modes

- when Webhook không khớp snapshot đơn (sai số tiền, đơn đã Paid hoặc không tồn tại): creditVerifiedWebhook trả false (không khớp trước giao dịch, hoặc PAYOS_WEBHOOK_MISMATCH trong callback làm rollback sổ cái), controller bỏ qua kết quả và vẫn trả received: true.
- when Webhook cùng orderCode được gửi lại: Ledger replay theo idempotencyKey, không cộng thêm kim cương; creditVerifiedWebhook trả false và controller vẫn trả received: true.
- when Lỗi khác khi ghi sổ hoặc cập nhật đơn: Ngoại lệ được ném lại, controller không trả received: true.
- when Client poll trước khi webhook payOS tới, hoặc webhook chưa được xử lý: Đơn vẫn Pending và transactionId null; client phải poll lại, endpoint không tự xác minh với payOS.
- when orderId không tồn tại hoặc thuộc user khác: findOneBy theo id và userId không thấy đơn nên ném lỗi không tìm thấy đơn, không lộ đơn của người khác.

## Endpoints covered

- POST /economy/payos/orders [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- POST /economy/webhooks/payos [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- GET /economy/payos/orders/:orderId [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
