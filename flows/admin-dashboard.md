# Bảng điều khiển admin

id: flow:admin-dashboard
group: Vận hành (admin)
services: module:admin, module:user, module:economy, module:party-room

Admin mở dashboard: dữ liệu nằm ở ba module khác nhau (người dùng, kinh tế, phòng party) cộng nhật ký audit, nên AdminService phải gom qua public API của từng module thay vì tự đọc bảng của họ.

AdminService gọi song song User, Economy, PartyRoom và audit, rồi tra nickname người thực hiện trước khi trả về dashboard.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- user (service): User
- economy (service): Economy
- party-room (service): Party Room
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/dashboard (role Admin/Moderator + quyền ViewUsers) (Controller đặt @RequireRoles(Admin, Moderator) ở cấp class, route thêm @RequireAdminPermission(ViewUsers); mọi route REST đều qua JwtAuthGuard.)
2. admin -> user [call/di-call]: Lấy số liệu người dùng: getAdminStats() (Bốn lời gọi nằm trong một Promise.all nên chạy song song.) {apps/core-api/src/modules/admin/admin.service.ts:635}
3. admin -> economy [call/di-call]: Lấy phân tích kim cương và VIP: getAdminAnalytics() {apps/core-api/src/modules/admin/admin.service.ts:636}
4. admin -> party-room [call/di-call]: Đếm phòng đang hoạt động: countActiveRooms() {apps/core-api/src/modules/admin/admin.service.ts:637}
5. admin -> postgres [call/shared-db]: Đọc 10 dòng audit gần nhất: listRecent(10) {apps/core-api/src/modules/admin/admin.service.ts:638}
6. admin -> user [call/di-call]: Tra nickname người thực hiện: findByIds(actorUserId) (Chạy sau Promise.all vì cần actorUserId từ kết quả audit.) {apps/core-api/src/modules/admin/admin.service.ts:640}
7. user -> admin [return/di-call]: Trả danh sách user để ghép nickname vào hoạt động
8. admin -> admin-console [return/http]: Trả dashboard gộp: người dùng, kim cương, phòng, hoạt động (Nếu không tìm thấy nickname thì dùng actorUserId; userTiers.free = max(0, activeUsers - VIP - SVIP).)

## Invariants

- AdminService không đọc bảng của module khác mà gọi UserService, EconomyService, PartyRoomService, vì mỗi module sở hữu dữ liệu của mình.
- Việc tra nickname phải chạy sau khi có danh sách audit, vì cần actorUserId của 10 dòng gần nhất.

## Failure modes

- when Một trong bốn lời gọi trong Promise.all bị lỗi: Promise.all từ chối nên getDashboard ném lỗi và dashboard không trả về.
- when Không tìm thấy nickname của người thực hiện: actorNickname rơi về actorUserId.

## Endpoints covered

- GET /admin/dashboard [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
