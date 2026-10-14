# Admin tra cứu người dùng và mở khoá

id: flow:admin-user-lookup-and-unban
group: Vận hành (admin)
services: module:admin, module:user

Admin xem danh sách, xem chi tiết rồi mở khoá một tài khoản. Module Admin chỉ điều phối: dữ liệu users thuộc User, còn bản ghi audit phải được ghi cùng transaction với việc đổi status để không có mở khoá nào thiếu dấu vết.

Admin-console gọi AdminController, AdminService chuyển việc đọc/đổi status cho UserService và ghi audit trong cùng một transaction.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/users: lọc theo status, role, nickname, phân trang (Controller yêu cầu role Admin hoặc Moderator và permission ViewUsers (AdminPermissionGuard).) {apps/core-api/src/modules/admin/admin.controller.ts:114}
2. admin -> user [call/di-call]: Chuyển bộ lọc sang UserService.findPage {apps/core-api/src/modules/admin/admin.service.ts:61}
3. user -> postgres [call/shared-db]: Truy vấn bảng users có lọc, sắp xếp createdAt giảm dần, skip/take {apps/core-api/src/modules/user/user.service.ts:194}
4. admin -> admin-console [return/http]: Trả trang danh sách người dùng
5. admin-console -> admin [call/http]: GET /admin/users/:id: xem hồ sơ nội bộ của một user {apps/core-api/src/modules/admin/admin.controller.ts:131}
6. admin -> user [call/di-call]: Gọi UserService.getByIdOrThrow {apps/core-api/src/modules/admin/admin.service.ts:53}
7. user -> postgres [call/shared-db]: Đọc user theo id, không thấy thì ném 404 {apps/core-api/src/modules/user/user.service.ts:88}
8. admin -> admin-console [return/http]: Trả AdminUserDto (status + role)
9. admin-console -> admin [call/http]: POST /admin/users/:id/unban: mở khoá tài khoản (Cần permission BanUsers.) {apps/core-api/src/modules/admin/admin.controller.ts:155}
10. admin -> postgres [call/shared-db]: Mở transaction cho mở khoá và ghi audit {apps/core-api/src/modules/admin/admin.service.ts:92}
11. admin -> user [call/di-call]: Gọi UserService.unbanUser với manager của transaction {apps/core-api/src/modules/admin/admin.service.ts:93}
12. user -> postgres [call/shared-db]: Đọc user rồi lưu status = Active {apps/core-api/src/modules/user/user.service.ts:255}
13. admin -> postgres [call/shared-db]: Ghi admin_audit_logs action user.unbanned cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:94, apps/core-api/src/common/audit/audit-log.service.ts:30}
14. admin -> admin-console [return/http]: Trả AdminUserDto của user đã mở khoá

## Invariants

- Admin không tự ghi bảng users: việc đổi status đi qua UserService (module sở hữu dữ liệu) nhưng dùng EntityManager của Admin, nên status và audit nằm trong cùng một transaction.
- Audit được ghi bằng cùng manager trong transaction, nên nếu ghi audit lỗi thì việc mở khoá cũng bị rollback, không có mở khoá thiếu dấu vết.

## Failure modes

- when User id không tồn tại (xem chi tiết hoặc mở khoá): UserService ném DomainException 404 PROFILE_NOT_FOUND; khi mở khoá, lỗi nổi ra khỏi callback nên transaction không ghi gì.
- when Ghi admin_audit_logs lỗi trong lúc mở khoá: Lỗi ném ra khỏi callback transaction, status của user không được lưu.

## Endpoints covered

- GET /admin/users [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/users/:id [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/users/:id/unban [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
