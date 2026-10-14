# Service map: Economy

id: service-map:economy
module: module:economy
deployable: core-api
confidence: high

Sổ cái kép, ví diamond, VIP, nạp tiền

Sở hữu sổ cái double-entry append-only (ledger_entries), snapshot ví wallets, catalog IAP/VIP/payOS, receipt và order nạp tiền; LedgerService là writer duy nhất vào ledger/wallet cho cả hệ thống. Không sở hữu hiệu ứng nghiệp vụ của nơi tiêu diamond (boost hàng đợi, ghi GiftEvent, cấp item avatar) và không phát realtime; các module khác chỉ gọi qua EconomyService trong index.ts.

Owns: bảng ledger_accounts, transactions (idempotency_key unique), ledger_entries (trigger chặn UPDATE/DELETE), wallets (snapshot balance/earnings/VIP); bảng iap_products, iap_receipts, vip_plans, payos_packages, payos_payment_orders; bảng outbox_events (relay -> Kafka topic litmatch.economy.events); advisory lock Postgres cho job đối soát (clusterSingleton)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/economy/wallet | jwt | Đọc ví của chính user: balance, earnings, tier VIP đã derive hết hạn khi đọc; chưa có ví thì trả 0. |
| e2 | rest | GET | /api/v1/economy/iap/products | jwt | Catalog gói diamond IAP đang bán (active=true) sắp theo diamonds tăng dần để client chọn productId trước khi verify. |
| e3 | rest | GET | /api/v1/economy/payos/packages | jwt | Catalog gói nạp payOS (packageId, amountVnd, diamonds) đang bán; client chỉ gửi packageId, giá luôn do server đọc. |
| e4 | rest | POST | /api/v1/economy/payos/orders | jwt | Tạo đơn nạp payOS idempotent (header Idempotency-Key, @Throttle 10/phút, guest bị 403): snapshot giá/diamond vào payos_payment_orders rồi gọi payOS lấy checkoutUrl/QR. |
| e5 | rest | GET | /api/v1/economy/payos/orders/:orderId | jwt | Trạng thái đơn payOS của chính user, nguồn sự thật phía server (returnUrl của browser không phải bằng chứng); đơn pending quá hạn bị chuyển expired khi đọc. |
| e6 | rest | GET | /api/v1/economy/vip/plans | jwt | Catalog gói VIP đang bán (planId, tier, số ngày, giá diamond) sắp theo giá tăng dần; client không hardcode bảng giá. |
| e7 | rest | POST | /api/v1/economy/iap/verify | jwt | Verify receipt IAP Apple/Google ở server rồi credit diamond; idempotency tự sinh iap:{provider}:{providerTransactionId}, @Throttle 10/phút, lưu iap_receipts cùng transaction. |
| e8 | rest | POST | /api/v1/economy/vip/purchase | jwt | Mua VIP bằng diamond (header Idempotency-Key): Nợ user_wallet / Có system_revenue và cập nhật vip_tier/vip_expires_at cộng dồn trong cùng DB transaction, kèm outbox economy.vip.purchased. |
| e9 | rest | GET | /api/v1/economy/transactions | jwt | Lịch sử giao dịch của chính user, cursor pagination (chưa gồm giao dịch chỉ là người nhận). |
| e10 | webhook | POST | /api/v1/economy/webhooks/apple | webhook-signature | Webhook App Store Server Notifications V2: verify JWS x5c lên Apple Root CA, đối chiếu bundleId, nếu REFUND/REVOKE thì reverse giao dịch credit; luôn ACK 200 sau khi verify. |
| e11 | webhook | POST | /api/v1/economy/webhooks/google/rtdn | webhook-signature | Webhook Google RTDN (Pub/Sub push): verify OIDC bearer token rồi chỉ log CANCELED làm tín hiệu phụ, KHÔNG refund (refund Google do job Voided Purchases đảm nhiệm). |
| e12 | webhook | POST | /api/v1/economy/webhooks/payos | webhook-signature | Webhook payOS: kiểm HMAC-SHA256 trước mọi query, đối chiếu orderCode/amount/currency/paymentLinkId với snapshot đơn rồi credit diamond và đánh dấu đơn paid trong một transaction. |
| e13 | job | WORKER | IapRefundPollService.onApplicationBootstrap | internal | Job quét refund (ECONOMY_REFUND_POLL_ENABLED): claim lô iap_receipts credited bằng FOR UPDATE SKIP LOCKED, hỏi Apple refund lookup / Google voided purchases rồi gọi RefundService; backstop cho webhook bị miss. |
| e14 | job | WORKER | OutboxRelayService.onApplicationBootstrap | internal | Outbox relay (ECONOMY_OUTBOX_RELAY_ENABLED): đọc outbox_events chưa publish theo lô FOR UPDATE SKIP LOCKED, publish Kafka, quá số lần thử thì đẩy dead-letter topic. |
| e15 | job | WORKER | ReconciliationService.onApplicationBootstrap | internal | Job đối soát hai tầng: fast kiểm tổng Nợ = tổng Có theo currency và receipt mồ côi, deep lấy mẫu ví so snapshot với ledger; read-only, chạy một replica nhờ advisory lock, lệch thì bắn metric. |
| e16 | job | WORKER | DevIapVerifier.onApplicationBootstrap | internal | Guard khi khởi động: ném lỗi nếu production vẫn cấu hình ECONOMY_IAP_VERIFIER=dev (verifier nhận receipt giả). |
| e17 | job | WORKER | DevAppleNotificationVerifier.onApplicationBootstrap | internal | Guard khi khởi động: ném lỗi nếu production vẫn dùng ECONOMY_APPLE_WEBHOOK_VERIFIER=dev (không verify chữ ký). |
| e18 | job | WORKER | DevGoogleRtdnVerifier.onApplicationBootstrap | internal | Guard khi khởi động: ném lỗi nếu production vẫn dùng ECONOMY_GOOGLE_RTDN_VERIFIER=dev (không verify OIDC). |

## Coordination edges (why this service talks to others)

### outbox -> Kafka economy (c1) with kafka [infra, out, kafka]

- endpoints: e14
- what: Relay publish outbox_events (economy.diamond.credited/debited/refunded, economy.vip.purchased, economy.gift.sent do sendGift ghi trong withinTransaction, economy.service.ts:645-660) lên topic litmatch.economy.events qua EventBusPort.publish; lỗi quá ECONOMY_OUTBOX_MAX_ATTEMPTS thì publishDeadLetter sang topic .DLQ.
- why: Sự kiện ví phải được ghi cùng DB transaction với bút toán (không mất, không phát khi rollback) nhưng Kafka không tham gia transaction Postgres, nên cần outbox + relay riêng để phản ứng phụ về sau không đọc thẳng bảng Economy. Hiện chưa có consumer nào của topic trong apps/ và relay mặc định tắt (env.validation.ts:349), nên đây là hợp đồng chuẩn bị sẵn.
- if missing: Nếu publish thẳng Kafka trong request thì rollback vẫn có event ma, hoặc commit xong mà Kafka lỗi thì mất event. Khi relay tắt (hosted profile) outbox chỉ tích lũy.
- evidence: apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:82, apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:117
- review: sound — Topic litmatch.economy.events, hậu tố .DLQ, economy.gift.sent (economy.service:650) và relay mặc định false đều đúng; không có consumer trong apps/.

### ledger/wallet trong Postgres (c2) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e12, e13, e14, e15
- what: Ghi ledger_entries, transactions, wallets, iap_receipts, payos_payment_orders, outbox_events trong một DB transaction; SELECT ... FOR UPDATE trên wallets; trigger DB chặn UPDATE/DELETE ledger_entries; job đối soát dùng pg advisory lock và SQL aggregate.
- why: Ledger là nguồn sự thật của tiền nên bất biến (Nợ = Có, idempotency_key unique, append-only) phải được DB cưỡng chế chứ không chỉ do code; snapshot wallets chỉ đúng khi cập nhật trong cùng transaction với entries và khóa dòng ví để tuần tự hóa per-user.
- if missing: Hai request song song cùng trừ tiền có thể làm âm ví hoặc credit hai lần một receipt; không có DB làm chốt chặn thì không đối soát và không audit được.
- evidence: apps/core-api/src/modules/economy/entities/iap.entities.ts:16, apps/core-api/src/modules/economy/entities/iap.entities.ts:40, apps/core-api/src/modules/economy/entities/ledger-account.entity.ts:24
- review: sound — lockWallet dùng pessimistic_write theo userId đã sort, trigger append-only ledger, unique receipt và advisory lock đối soát đều có trong code.

### refund/tra cứu ví cho admin (c3) with admin [service, in, di-call]

- endpoints: 
- what: AdminService gọi EconomyService.getWallet, listTransactions, adminRefundTransaction(transactionId, reason, actorUserId, withinTransaction), listCatalogForAdmin, setIapProductActive/setVipPlanActive, listActiveVipUserIds, getAdminAnalytics.
- why: Hoàn tiền thủ công phải là bút toán đảo qua ledger.reverse của chính Economy (writer duy nhất), và audit log của admin phải commit cùng transaction với reversal qua callback withinTransaction, nếu không có thể hoàn tiền mà không có dấu vết hoặc ngược lại.
- if missing: Admin phải tự sửa bảng ledger/wallet (bị trigger chặn và vi phạm quy tắc writer duy nhất) hoặc refund và audit tách transaction nên có thể lệch nhau.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:313, apps/core-api/src/modules/admin/admin.service.ts:326, apps/core-api/src/modules/admin/admin.service.ts:344
- review: sound — Trùng cơ chế admin.c1: reverse với key admin-refund:{id}, callback withinTransaction chạy cùng manager sau khi đánh dấu giao dịch gốc Reversed.

### trừ diamond mua item avatar (c4) with avatar [service, in, di-call]

- endpoints: 
- what: AvatarService.buy gọi EconomyService.spendDiamond(userId, AvatarPurchase, priceDiamond, idempotencyKey, metadata) rồi mới ghi quyền sở hữu item.
- why: Giá item lấy từ catalog avatar nhưng việc trừ ví phải qua writer duy nhất của ledger để có khóa ví, kiểm số dư tại thời điểm trừ và idempotency; spendDiamond không có hook transaction nên Avatar phải spend trước rồi grant idempotent (retry cùng key an toàn).
- if missing: Avatar phải tự ghi ví (cấm) hoặc item được cấp mà không trừ diamond; mua lặp do retry sẽ trừ hai lần nếu thiếu idempotency key.
- evidence: apps/core-api/src/modules/avatar/avatar.service.ts:85
- review: sound — spendDiamond không có hook transaction (comment avatar.service:68-70); replay trả replayed rồi grantOwnership ON CONFLICT DO NOTHING.

### sendGift hai chân DIA/PTS (c5) with gift [service, in, di-call]

- endpoints: 
- what: GiftService.sendGift (3 điểm gọi: quà phòng, video, profile) gọi EconomyService.sendGift({senderUserId, receiverUserId, priceDiamond, pointsAwarded, idempotencyKey, metadata, withinTransaction}); callback ghi GiftEvent và Notification (và Conversation ở profile) bằng manager của ledger, Economy tự ghi thêm outbox economy.gift.sent trong cùng transaction.
- why: Quà là một transaction hai chân độc lập theo currency (người tặng mất DIA, người nhận được PTS theo GIFT_POINTS_RATE_PERCENT, 0 nếu guest). Callback chạy sau khi entries và wallet delta đã ghi, trong cùng EntityManager (ledger.service.ts:179) nên trừ/cộng, GiftEvent, Notification cùng commit hoặc cùng rollback; replay cùng key trả sớm và không chạy lại callback nên không ghi đôi.
- if missing: Hỏng một bước sẽ để lại ví đã trừ mà người nhận không có điểm/event, hoặc điểm PTS được mint mà người tặng không mất DIA.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:210, apps/core-api/src/modules/gift/gift.service.ts:308, apps/core-api/src/modules/gift/gift.service.ts:401
- review: sound — recordInManager gọi withinTransaction sau entries và applyWalletDeltas (ledger.service:179), replay trả sớm trước callback; chân PTS chỉ có khi points>0.

### trừ diamond match, đọc VIP (c6) with matching [service, in, di-call]

- endpoints: 
- what: MatchingService gọi spendDiamondInTransaction(manager, ...) cho lượt match mua thêm, spendDiamond cho speed-up, và getActiveVipTier(userId) để chọn giá speed-up VIP/SVIP.
- why: Extra match trả phí trừ diamond trong CÙNG transaction với MatchTicket nhờ spendDiamondInTransaction nhận manager của Matching (matching.service.ts:139) nên rollback cùng nhau. Speed-up thì KHÔNG: spendDiamond commit riêng rồi mới tăng priorityBoostMs, và boost chỉ chạy khi !spend.replayed (matching.service.ts:448-466), nên crash giữa hai bước làm retry cùng key thấy replayed=true và bỏ qua boost. Tier VIP chỉ Economy biết (getActiveVipTier) nên giá VIP/SVIP phải hỏi tại lúc hành động.
- if missing: Extra match: ticket tạo mà không bị trừ tiền (hoặc ngược lại). Speed-up: có cửa sổ trừ tiền mà không có boost; giá VIP phải dựa vào bản sao tier cũ ở Matching.
- evidence: apps/core-api/src/modules/matching/matching.service.ts:139, apps/core-api/src/modules/matching/matching.service.ts:264, apps/core-api/src/modules/matching/matching.service.ts:448
- review: sound — Extra match dùng recordInManager cùng manager; speed-up spendDiamond commit riêng, increment boost chỉ khi !spend.replayed (matching.service:448-466).

### verify receipt, tra refund (c7) with external:apple [external, out, http]

- endpoints: e7, e13
- what: Gọi Apple verifyReceipt (production rồi sandbox) để xác minh receipt khi nạp, và App Store Server API /inApps/v2/refund/lookup/{transactionId} (JWT ES256) để job quét kiểm refund từng giao dịch.
- why: Chỉ Apple biết receipt có thật và đã bị hoàn hay chưa; client không đáng tin nên diamond chỉ được credit sau khi server tự hỏi Apple, và refund phải được dò chủ động vì webhook không đảm bảo 100%.
- if missing: Client có thể gửi receipt giả để nhận diamond; refund Apple bị miss webhook sẽ giữ diamond đã hoàn tiền mà không ai thu hồi.
- evidence: apps/core-api/src/modules/economy/clients/apple-receipt-api.adapter.ts:100, apps/core-api/src/modules/economy/clients/apple-refund-api.adapter.ts:29
- review: sound — Production rồi sandbox đúng, nhưng sandbox chỉ gọi khi status là sandbox-receipt; refund lookup dùng JWT ES256 (apple-server-api.ts).

### Server Notification REFUND (c8) with external:apple [external, in, webhook]

- endpoints: e10
- what: Apple POST signedPayload (JWS) tới /economy/webhooks/apple; server verify chain x5c lên Apple Root CA (ECONOMY_APPLE_ROOT_CA_PEM), đối chiếu bundleId với ECONOMY_APPLE_BUNDLE_ID (bỏ qua nếu biến rỗng, mặc định rỗng), rồi RefundService.refundIapPurchase khi REFUND/REVOKE.
- why: Apple cho user hoàn tiền sau khi đã nạp và có thể đã tiêu diamond; chỉ tín hiệu server-to-server đã verify chữ ký mới được kích hoạt bút toán đảo, và endpoint @Public nên chữ ký là lớp xác thực duy nhất.
- if missing: Không biết refund kịp thời nên user giữ diamond free; hoặc nếu không verify chữ ký thì kẻ ngoài giả notification để đảo giao dịch hợp lệ của người khác.
- evidence: apps/core-api/src/modules/economy/ports/notification-verifier.ts:107
- review: corrected — Đối chiếu bundleId chỉ chạy khi ECONOMY_APPLE_BUNDLE_ID khác rỗng (mặc định rỗng); không phải luôn luôn.

### purchase token + voided (c9) with external:google-play [external, out, http]

- endpoints: e7, e13
- what: Gọi Android Publisher API (OAuth service account) purchases/products/.../tokens/{purchaseToken} để verify purchase khi nạp và purchases/voidedpurchases để liệt kê giao dịch bị void cho job quét.
- why: Google không có refund webhook đáng tin cho one-time product (CANCELED không đồng nghĩa voided), nên Voided Purchases API là nguồn refund chính thức duy nhất; verify purchase token cũng chỉ Google xác nhận được.
- if missing: Purchase token giả được credit, và refund Google one-time không bao giờ bị thu hồi diamond.
- evidence: apps/core-api/src/modules/economy/clients/google-play-receipt-api.adapter.ts:45, apps/core-api/src/modules/economy/clients/google-refund-api.adapter.ts:37
- review: sound — Doc economy-service § 5 nêu Voided Purchases là nguồn refund chính thức cho one-time; RTDN chỉ log CANCELED, không refund.

### RTDN Pub/Sub push (c10) with external:google-play [external, in, webhook]

- endpoints: e11
- what: Google Pub/Sub push RTDN tới /economy/webhooks/google/rtdn kèm Bearer OIDC token; server verify issuer, audience, email service account (ECONOMY_GOOGLE_RTDN_*) rồi chỉ log notificationType=2 (CANCELED).
- why: Endpoint @Public nên OIDC token là cách duy nhất chứng minh push đến từ subscription đã cấu hình; không refund tại đây vì CANCELED không đảm bảo nghĩa voided với one-time product (doc § 5).
- if missing: Không verify thì ai cũng POST được vào endpoint; nếu refund theo CANCELED thì đảo nhầm giao dịch hợp lệ.
- evidence: apps/core-api/src/modules/economy/ports/notification-verifier.ts:251
- review: sound — jwtVerify kiểm issuer, audience, email và email_verified; controller chỉ log notificationType 2 (CANCELED), không refund.

### tạo payment link VietQR (c11) with external:payos [external, out, http]

- endpoints: e4
- what: PayosClient.createPaymentLink POST {PAYOS_API_BASE_URL}/v2/payment-requests (header x-client-id/x-api-key, HMAC-SHA256 trong JSON body); khi timeout, lỗi mạng hoặc phản hồi không hợp lệ thì GET /v2/payment-requests/{orderCode} lấy paymentLinkId, checkoutUrl dựng lại từ id (qrCode null).
- why: Checkout URL/QR chỉ payOS cấp được; vì POST có thể đã tạo link nhưng response mất, phải tra lại theo orderCode (unique trong DB) để retry không sinh checkout thứ hai cho cùng intent.
- if missing: Web không có link/QR VietQR để user thanh toán VND; retry sau timeout tạo nhiều link cho cùng đơn.
- evidence: apps/core-api/src/modules/economy/ports/payos-client.ts:80
- review: corrected — GET khôi phục cũng chạy khi phản hồi không hợp lệ, không chỉ timeout; link dựng lại từ id, qrCode null.

### webhook thanh toán payOS (c12) with external:payos [external, in, webhook]

- endpoints: e12
- what: payOS POST /economy/webhooks/payos {code, success, data, signature}; PayosClient.verifyWebhook tính lại HMAC-SHA256 bằng PAYOS_CHECKSUM_KEY rồi PayosService.creditVerifiedWebhook ghi ledger (Nợ system_iap / Có user_wallet) và đặt đơn paid cùng transaction.
- why: Chỉ webhook có HMAC hợp lệ mới là bằng chứng đã trả tiền (returnUrl của browser thì không); endpoint @Public nên chữ ký là lớp xác thực duy nhất, và idempotency key payos-credit:{orderCode} chặn cộng diamond lần hai khi payOS retry.
- if missing: Không có webhook thì diamond không bao giờ được cộng sau khi trả tiền; không kiểm HMAC thì ai cũng giả được thanh toán để nhận diamond free.
- evidence: apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:125, apps/core-api/src/modules/economy/ports/payos-client.ts:295
- review: sound — creditVerifiedWebhook ghi Nợ system_iap/Có user_wallet, key payos-credit:{orderCode}, lockOrder và đặt Paid trong withinTransaction; so chữ ký bằng timingSafeEqual.

### Idempotency-Key, hỏi đơn (c13) with web [client, both, http]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9
- what: Web gửi header Idempotency-Key cho POST /economy/payos/orders và /economy/vip/purchase, chuyển user sang checkoutUrl của payOS, quay về với ?paymentOrder=<orderId> rồi GET /economy/payos/orders/:orderId (nút 'Kiểm tra lại') và invalidate ví khi paid.
- why: Idempotency-Key do client sinh là cách duy nhất để retry mạng không tạo đơn/mua VIP thứ hai; trạng thái paid chỉ biết sau khi webhook ghi ledger nên web phải hỏi lại server thay vì tin redirect.
- if missing: Bấm đúp hoặc retry tạo hai đơn/trừ diamond hai lần; web tin returnUrl sẽ hiện 'thành công' khi chưa có webhook.
- evidence: apps/web/src/features/wallet/api.ts:76, apps/web/src/features/wallet/components/payos-payment-status.tsx:12
- review: sound — Web gửi Idempotency-Key cho payos/orders và vip/purchase, có nút 'Kiểm tra lại', invalidate ví khi status paid.

## Notes

- Doc economy-service.md § 3 mô tả idempotency đồng thời bằng 'retry đọc ngắn có backoff'; code (ledger.service.ts recordInManager) dùng INSERT ... ON CONFLICT DO NOTHING rồi đọc lại row thắng trong cùng transaction, và record() bắt unique violation rồi đọc winner. Không có vòng backoff.
- Doc § 4 nói payload event là {version, transactionId, userId, amount, transactionType}; riêng economy.vip.purchased thực tế là {version, userId, planId, tier, vipExpiresAt} (economy.service.ts purchaseVip), không có transactionId/amount.
- Doc § 2 nói có rate-limit verify theo user; code chỉ có @Throttle 10 request/phút trên /iap/verify và /payos/orders qua ThrottlerGuard, app.module.ts không cấu hình storage nên là bộ đếm in-memory mặc định theo từng instance (tracker mặc định của Nest là IP), không phải theo user và không chia sẻ giữa các replica.
- Doc § 7 nhắc StoreKit 2 / App Store Server API, nhưng xác minh receipt Apple khi nạp dùng endpoint legacy verifyReceipt (economy.constants.ts APPLE_VERIFY_RECEIPT_URL); App Store Server API chỉ dùng cho refund lookup của job quét.
- Job đối soát deep chỉ so snapshot với ledger cho account user_wallet (DIA); wallets.earnings (PTS) không được kiểm, dù doc § 1.2 nói cả hai rebuild được.
- ECONOMY_OUTBOX_RELAY_ENABLED và ECONOMY_REFUND_POLL_ENABLED mặc định false (env.validation.ts) và hosted profile (deploy/hosted/core-api.env.example) cũng tắt: outbox_events vẫn được ghi nhưng published_at giữ null, và không còn backstop refund ngoài webhook Apple. Không tìm thấy consumer nào của topic litmatch.economy.events trong apps/ — event hiện là hợp đồng tương lai.
- Webhook Google RTDN chỉ log tín hiệu CANCELED, không refund (đúng doc); refund Google one-time chỉ xảy ra qua IapRefundPollService, nên khi ECONOMY_REFUND_POLL_ENABLED=false thì refund Google không bao giờ được xử lý.
- GET /economy/payos/orders/:orderId có side effect ghi: đơn pending quá hạn bị UPDATE sang expired ngay khi đọc (payos.service.ts getOrderStatus); webhook paid đến trễ vẫn thắng vì creditVerifiedWebhook ghi đè status về paid.
- avatar.buy gọi spendDiamond rồi mới grantOwnership ở transaction khác (avatar.service.ts:85-92); không atomic, an toàn nhờ retry idempotent cùng key. Khác Gift/Matching extra match dùng cùng DB transaction.
- Các webhook payOS có orderCode lạ hoặc sai số tiền/paymentLinkId vẫn được ACK 200 và không ghi sổ (creditVerifiedWebhook trả false); chỉ log, không alert.
- ledger.recordInManager trả sớm với replayed=true khi idempotencyKey đã tồn tại (ledger.service.ts:106-114) nên withinTransaction (GiftEvent, outbox, VIP) KHÔNG chạy lại ở replay; caller phải đọc lại kết quả theo transactionId (Gift đọc GiftEvent qua transaction_id).
- Matching speed-up có cửa sổ mất boost: spendDiamond commit riêng rồi ticketRepo.increment chỉ chạy khi !spend.replayed (matching.service.ts:448-466); crash giữa hai bước thì retry cùng key thấy replayed=true và bỏ qua boost (tiền đã trừ, boost mất). Khác Avatar, nơi grantOwnership chạy vô điều kiện sau spend (avatar.service.ts:85-92). Đã gửi yêu cầu sang matching.
