# Admin gửi thông báo hàng loạt

id: flow:admin-broadcast-notification
group: Vận hành (admin)
services: module:admin, module:user, module:economy, module:notification

Admin broadcast thông báo tới mọi user active: Admin điều phối, User cấp danh sách người nhận, Economy cấp danh sách VIP để lọc audience, Notification ghi thông báo theo lô rồi đẩy push, vì mỗi module chỉ được đọc/ghi dữ liệu của mình.

Admin lấy người nhận từ User/Economy, ghi thông báo và audit trong một transaction, rồi mới đẩy push best-effort qua Notification.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- user (service): User
- economy (service): Economy
- notification (service): Notification
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: POST /admin/notifications/broadcast (title, body, audience) (Route cần quyền ManageConfig qua @RequireAdminPermission.) {apps/core-api/src/modules/admin/admin.controller.ts:490}
2. admin -> user [call/di-call]: Lấy danh sách id user đang active {apps/core-api/src/modules/admin/admin.service.ts:424}
3. user -> postgres [call/shared-db]: SELECT id FROM users WHERE status = Active (loại banned) {apps/core-api/src/modules/user/user.service.ts:113}
4. admin -> economy [call/di-call]: Lấy id user VIP còn hạn (chỉ khi audience không phải All) {apps/core-api/src/modules/admin/admin.service.ts:428}
5. economy -> postgres [call/shared-db]: SELECT userId FROM wallets WHERE vipExpiresAt > now {apps/core-api/src/modules/economy/economy.service.ts:225}
6. admin -> notification [call/di-call]: Mở transaction, gọi tạo thông báo broadcast cho recipientIds (recipientIds được lọc trong Admin theo audience (All/Vip/không VIP).) {apps/core-api/src/modules/admin/admin.service.ts:438}
7. notification -> postgres [call/shared-db]: INSERT notifications loại AdminBroadcast, chia lô 500 bản ghi (Dùng manager của transaction nên cùng transaction với audit.) {apps/core-api/src/modules/notification/notification.service.ts:85}
8. admin -> postgres [call/shared-db]: Ghi audit notification.broadcast.sent cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:443}
9. admin -> notification [async/di-call]: Sau commit: đẩy push cho từng thông báo đã tạo (sendPush bọc try/catch, lỗi chỉ log cảnh báo, không throw.) {apps/core-api/src/modules/admin/admin.service.ts:463}
10. admin -> admin-console [return/http]: Trả broadcastId và recipientCount

## Invariants

- Admin phải lấy id từ User và Economy qua service của họ, không tự đọc bảng users hay wallets, để mỗi module giữ quyền sở hữu dữ liệu.
- Thông báo in-app và audit log nằm cùng một transaction: broadcast chỉ được ghi nếu có dấu vết audit, và ngược lại.
- Push chỉ chạy sau khi transaction commit, nên không thể gửi push cho thông báo chưa được lưu.

## Failure modes

- when Ghi thông báo hoặc audit lỗi trong transaction: Transaction rollback, không có thông báo nào được lưu và không có push nào được gửi.
- when Push provider hoặc web push lỗi: sendPush nuốt lỗi và chỉ log cảnh báo; thông báo in-app đã commit vẫn được giữ.
- when Không có người nhận (recipientIds rỗng): createBroadcastWithManager trả mảng rỗng, không ghi thông báo, không gửi push.

## Endpoints covered

- POST /admin/notifications/broadcast [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
