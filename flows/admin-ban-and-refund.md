# Admin khoá tài khoản và hoàn giao dịch

id: flow:admin-ban-and-refund
group: Vận hành (admin)
services: module:admin, module:user, module:economy

Admin khoá một người dùng và hoàn một giao dịch diamond. AdminModule chỉ điều phối: dữ liệu users do user sở hữu, sổ cái và ví do economy sở hữu, nên mọi thay đổi phải đi qua service chủ dữ liệu và audit log phải ghi cùng transaction với hành động.

Hai luồng back-office: ban user và refund giao dịch, mỗi luồng ghi audit log cùng một DB transaction với thay đổi dữ liệu.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- user (service): User
- economy (service): Economy
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: POST /admin/users/:id/ban (Route yêu cầu quyền AdminPermission.BanUsers (RequireAdminPermission), ngoài JWT guard toàn cục.)
2. admin -> user [call/di-call]: Mở transaction rồi gọi UserService.banUser(manager, targetUserId) (AdminService chặn tự ban chính mình trước khi mở transaction (actorUserId === targetUserId thì trả 403).) {apps/core-api/src/modules/admin/admin.service.ts:77}
3. user -> postgres [call/shared-db]: Đọc user và lưu status = Banned bằng manager của admin {apps/core-api/src/modules/admin/admin.service.ts:76}
4. admin -> postgres [call/shared-db]: Ghi audit log user.banned cùng manager {apps/core-api/src/modules/admin/admin.service.ts:78}
5. admin -> admin-console [return/http]: Trả về user đã bị khoá
6. admin-console -> admin [call/http]: POST /admin/economy/transactions/:id/refund kèm lý do (Route yêu cầu quyền AdminPermission.RefundTransaction (RequireAdminPermission).)
7. admin -> economy [call/di-call]: Gọi adminRefundTransaction kèm callback ghi audit (AdminService không mở transaction riêng; economy giữ vai trò writer duy nhất của ledger. EconomyService.adminRefundTransaction gọi ledger.reverse với key admin-refund:<transactionId> và outbox override economy.diamond.refunded.) {apps/core-api/src/modules/admin/admin.service.ts:344}
8. economy -> postgres [call/shared-db]: ledger.record: transaction mở bút toán đảo, cập nhật ví, ghi outbox refunded (ledger.reverse đọc giao dịch gốc và kiểm Reversed (409 TRANSACTION_ALREADY_REVERSED) TRƯỚC khi mở transaction; trong transaction ghi entry đảo chiều, khoá và cập nhật ví, ghi outbox (chỉ khi balanceDelta khác 0) rồi chạy withinTransaction. Reversal được phép đẩy balance âm.) {apps/core-api/src/modules/economy/services/ledger.service.ts:179}
9. economy -> postgres [call/shared-db]: Đánh dấu giao dịch gốc là Reversed (UPDATE có điều kiện status) {apps/core-api/src/modules/economy/services/ledger.service.ts:258}
10. economy -> admin [call/di-call]: Gọi lại callback withinTransaction(manager) của admin {apps/core-api/src/modules/admin/admin.service.ts:348}
11. admin -> postgres [call/shared-db]: Ghi audit log economy.transaction.refunded kèm reason cùng manager {apps/core-api/src/modules/admin/admin.service.ts:349}
12. economy -> admin [return/di-call]: Trả transactionId và reversalTransactionId
13. admin -> admin-console [return/http]: Phản hồi kết quả hoàn tiền

## Invariants

- Ban và audit log nằm trong cùng một dataSource.transaction, nên không thể đổi status mà thiếu audit hoặc ngược lại.
- UserService.banUser nhận manager từ AdminService thay vì tự mở transaction, để dữ liệu của user vẫn do user ghi nhưng cùng ranh giới atomic với audit.
- Refund đi qua EconomyService, và audit được truyền xuống thành callback chạy trong transaction của ledger.reverse; economy là writer duy nhất vào ledger và ví.
- Idempotency key tất định theo transactionId và UPDATE có điều kiện status chặn hoàn kép; nếu mất race thì ném lỗi và toàn bộ bút toán đảo rollback.

## Failure modes

- when Tự ban chính mình (actorUserId bằng targetUserId): Ném CANNOT_BAN_SELF 403 trước khi mở transaction, không có thay đổi nào.
- when Không tìm thấy user cần ban: setStatus ném PROFILE_NOT_FOUND 404, transaction rollback, không có audit log.
- when Ghi audit log lỗi trong lúc ban hoặc refund: Lỗi lan ra khỏi callback trong transaction, nên thay đổi status hoặc bút toán đảo cùng rollback.
- when Giao dịch gốc đã được hoàn: ledger.reverse trả 409 TRANSACTION_ALREADY_REVERSED; nếu thua race lúc đánh dấu thì rollback bút toán đảo của request đó.
- when Hoàn tiền làm điểm earnings của user âm: applyWalletDeltas ném WALLET_INSUFFICIENT_BALANCE (422) vì newEarnings < 0 (chỉ balance mới được phép âm với Reversal), transaction rollback, không có audit log.

## Endpoints covered

- POST /admin/users/:id/ban [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/economy/transactions/:id/refund [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
