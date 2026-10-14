# Admin quản lý nhân sự và ma trận quyền

id: flow:admin-staff-and-permissions
group: Vận hành (admin)
services: module:admin, module:user

Admin xem và bật/tắt quyền theo vai trò, liệt kê nhân sự và đổi vai trò. AdminModule giữ ma trận quyền và audit, còn bảng users thuộc module user, nên đổi vai trò phải đi qua UserService trong cùng một transaction với audit.

Guard đọc lại vai trò và quyền từ DB mỗi request; đổi quyền hoặc vai trò ghi audit cùng transaction.

## Participants

- admin-console (client): Admin console
- admin (service): Admin API
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/permissions xem ma trận quyền (AdminPermissionGuard gắn ở class AdminController và chạy trước mỗi route; cả 4 route đều yêu cầu ManagePermissions.)
2. admin -> user [call/di-call]: Guard đọc lại role hiện tại của người gọi (Vì đọc từ DB nên hạ quyền có hiệu lực ngay dù access token cũ còn hạn.) {apps/core-api/src/modules/admin/admin.service.ts:473}
3. admin -> postgres [call/shared-db]: Guard tra chính sách enabled của role/permission {apps/core-api/src/modules/admin/admin.service.ts:476}
4. admin -> postgres [call/shared-db]: Đọc toàn bộ admin_role_permissions dựng ma trận {apps/core-api/src/modules/admin/admin.service.ts:491}
5. admin -> admin-console [return/http]: Trả ma trận quyền moderator/admin
6. admin-console -> admin [call/http]: PATCH /admin/permissions/:role/:permission (Guard lại đọc role và chính sách như ở bước 2-3 trước khi vào service.)
7. admin -> postgres [call/shared-db]: Cập nhật cờ enabled của chính sách trong transaction (Trả 404 nếu không có dòng chính sách.) {apps/core-api/src/modules/admin/admin.service.ts:550}
8. admin -> postgres [call/shared-db]: Ghi admin_audit_logs cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:551}
9. admin -> admin-console [return/http]: 204 No Content sau khi commit
10. admin-console -> admin [call/http]: GET /admin/staff liệt kê nhân sự
11. admin -> user [call/di-call]: Nhờ UserService lấy user có role staff {apps/core-api/src/modules/admin/admin.service.ts:565}
12. user -> postgres [call/shared-db]: Tìm users role moderator hoặc admin, sắp theo createdAt {apps/core-api/src/modules/user/user.service.ts:119}
13. admin -> admin-console [return/http]: Trả danh sách nhân sự
14. admin-console -> admin [call/http]: PATCH /admin/staff/:id/role đổi vai trò (Từ chối nếu actor tự đổi role của chính mình (CANNOT_CHANGE_OWN_ROLE) trước khi mở transaction.)
15. admin -> user [call/di-call]: Khoá admin hiện tại + target, đếm số admin {apps/core-api/src/modules/admin/admin.service.ts:581}
16. user -> postgres [call/shared-db]: SELECT users role admin hoặc id target FOR UPDATE (Hai request hạ quyền song song phải xếp hàng nên không thể cùng hạ hai admin cuối.) {apps/core-api/src/modules/user/user.service.ts:163}
17. admin -> user [call/di-call]: Chặn hạ admin cuối, rồi gán role mới cho target {apps/core-api/src/modules/admin/admin.service.ts:597}
18. user -> postgres [call/shared-db]: Lưu user với role mới bằng manager của transaction {apps/core-api/src/modules/user/user.service.ts:185}
19. admin -> postgres [call/shared-db]: Ghi audit previousRole và role mới cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:602}
20. admin -> admin-console [return/http]: Trả AdminStaffDto của user đã đổi role

## Invariants

- Guard đọc role từ DB (UserService) rồi tra admin_role_permissions mỗi request, nên hạ quyền hoặc tắt quyền có hiệu lực ngay dù JWT cũ còn hạn.
- Ghi audit dùng cùng manager với thay đổi quyền/role: nếu audit lỗi thì thay đổi rollback, không có thay đổi quyền nào thiếu dấu vết.
- Bảng users thuộc module user nên admin phải gọi UserService với manager chung của transaction, thay vì tự ghi bảng users.
- Khoá pessimistic_write trên mọi admin và target trước khi đếm adminCount để kiểm tra last-admin không bị race giữa hai request song song.

## Failure modes

- when Người gọi không có quyền ManagePermissions hoặc route thiếu metadata quyền: Guard ném PERMISSION_FORBIDDEN (403), service không chạy.
- when Admin tắt ManagePermissions của role admin: Ném CANNOT_DISABLE_PERMISSION_CONTROL (409) trước khi mở transaction.
- when Actor tự đổi role của mình: Ném CANNOT_CHANGE_OWN_ROLE (409), không mở transaction.
- when Hạ vai trò admin cuối cùng (adminCount <= 1): Ném CANNOT_DEMOTE_LAST_ADMIN (409) trong transaction, rollback, không đổi gì.
- when Ghi audit lỗi trong transaction: Cả thay đổi quyền/role và audit rollback.

## Endpoints covered

- GET /admin/permissions [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/permissions/:role/:permission [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/staff [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/staff/:id/role [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
