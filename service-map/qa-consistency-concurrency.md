# Nhất quán và đồng thời: transaction, khoá, sổ cái?

id: qa:consistency-concurrency
category: consistency
services: module:economy, module:gift, module:soul-match, module:movie-match, module:matching

Một PostgreSQL chung nên atomicity xuyên module dùng chung một transaction (truyền EntityManager), không saga hay 2PC. Race được chặn bằng khoá bi quan (SELECT FOR UPDATE), advisory lock và ràng buộc unique làm chốt cuối; tiền đi qua sổ cái kép append-only và sửa sai bằng bút toán đảo.

### Chung transaction

Gift gọi Economy kèm EntityManager của giao dịch đang chạy nên trừ tiền người tặng, cộng người nhận và ghi GiftEvent cùng commit hoặc cùng rollback.

services: gift, economy; evidence: apps/core-api/src/modules/gift/gift.service.ts:104

### Khoá chống race

Ledger khoá ví rồi mới ghi; soul-match khoá hàng match_sessions để hai rating like song song không cùng kết luận chưa đủ hai like; movie-match dùng advisory xact lock cho matcher ẩn danh.

services: economy, soul-match, movie-match; evidence: apps/core-api/src/modules/economy/services/ledger.service.ts:420, apps/core-api/src/modules/soul-match/soul-match.service.ts:165, apps/core-api/src/modules/movie-match/movie-match.service.ts:680

### Sổ cái kép

Mỗi giao dịch tiền tạo ít nhất hai LedgerEntry nợ và có, append-only; Wallet.balance chỉ là snapshot cập nhật trong cùng transaction; correction là reversal.

services: economy; evidence: apps/core-api/src/modules/economy/economy.module.ts:58

## Alternatives not chosen

- Saga hoặc outbox giữa các module: Chỉ cần khi tách thành deployable riêng (docs/03 § 3.6); trong cùng process thì thừa.
- 2PC xuyên deployable: docs/03 § 3.6 chủ động không dùng 2PC xuyên deployable.
