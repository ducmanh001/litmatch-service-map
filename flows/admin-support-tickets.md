# Người dùng gửi ticket hỗ trợ, admin xử lý

id: flow:admin-support-tickets
group: Vận hành (admin)
services: module:support, module:admin

Người dùng gửi ticket kèm Idempotency-Key để không bị tạo trùng khi gọi lại, rồi xem lại ticket của mình. Support chỉ có endpoint cho người dùng, nên staff xử lý qua module Admin: Admin kiểm quyền, gọi Support đổi trạng thái và ghi audit log cùng một transaction.

Ticket được tạo idempotent ở Support, còn việc admin liệt kê và đổi trạng thái đi qua Admin để kiểm quyền và ghi audit atomically.

## Participants

- web (client): Web client
- admin-console (client): Admin console
- support (service): Support
- admin (service): Admin
- postgres (infra): PostgreSQL

## Steps

1. web -> support [call/http]: POST /support/tickets kèm header Idempotency-Key (Mọi route REST đều qua JwtAuthGuard nên userId lấy từ token.) {apps/core-api/src/modules/support/support.controller.ts:48}
2. support -> postgres [call/shared-db]: Lưu support_tickets với status open và idempotencyKey (Nếu vi phạm ràng buộc unique uq_support_ticket_user_idempotency thì đọc lại ticket cũ (findOneByOrFail) và trả lại bản đó; nếu category hoặc message khác thì ném IDEMPOTENCY_CONFLICT (409).) {apps/core-api/src/modules/support/support.service.ts:36}
3. support -> web [return/http]: Trả 201 với ticket vừa tạo (hoặc bản replay)
4. web -> support [call/http]: GET /support/tickets/me để theo dõi ticket của mình {apps/core-api/src/modules/support/support.controller.ts:63}
5. support -> postgres [call/shared-db]: Query ticket theo userId, phân trang cursor {apps/core-api/src/modules/support/support.service.ts:111}
6. support -> web [return/http]: Trả trang ticket của người dùng
7. admin-console -> admin [call/http]: GET /admin/support/tickets (lọc status, cursor) (Endpoint yêu cầu quyền ManageSupport.) {apps/core-api/src/modules/admin/admin.controller.ts:321}
8. admin -> support [call/di-call]: Gọi SupportService.listAll(limit, cursor, status) {apps/core-api/src/modules/admin/admin.service.ts:242}
9. support -> postgres [call/shared-db]: Query mọi ticket, lọc theo status nếu có {apps/core-api/src/modules/support/support.service.ts:111}
10. admin -> admin-console [return/http]: Trả trang ticket cho moderator
11. admin-console -> admin [call/http]: PATCH /admin/support/tickets/:id đổi status, staffResponse {apps/core-api/src/modules/admin/admin.controller.ts:340}
12. admin -> postgres [call/shared-db]: Mở transaction cho cập nhật ticket {apps/core-api/src/modules/admin/admin.service.ts:250}
13. admin -> support [call/di-call]: Gọi SupportService.setStatusWithManager(manager, ...) {apps/core-api/src/modules/admin/admin.service.ts:251}
14. support -> postgres [call/shared-db]: Khóa dòng ticket, kiểm tra chuyển trạng thái, lưu status mới (Không tìm thấy ticket thì TICKET_NOT_FOUND (404); chuyển trạng thái không hợp lệ thì INVALID_TRANSITION (409).) {apps/core-api/src/modules/support/support.service.ts:83}
15. admin -> postgres [call/shared-db]: Ghi admin_audit_logs support.ticket.updated cùng manager {apps/core-api/src/modules/admin/admin.service.ts:256}
16. admin -> admin-console [return/http]: Trả ticket đã cập nhật sau khi commit

## Invariants

- Idempotency-Key được ràng buộc unique theo (userId, key) ở bảng support_tickets, nên gọi lại cùng key trả lại ticket cũ thay vì tạo trùng.
- Support không có endpoint cho staff; mọi thao tác admin phải đi qua AdminService để kiểm quyền ManageSupport và để audit log nằm cùng transaction với việc đổi trạng thái.
- setStatusWithManager nhận manager từ Admin và khóa dòng ticket (pessimistic_write), nên hai admin đổi trạng thái đồng thời được tuần tự hóa.

## Failure modes

- when Gọi lại với cùng Idempotency-Key nhưng category hoặc message khác: Support ném IDEMPOTENCY_CONFLICT với HTTP 409, không tạo ticket mới.
- when Chuyển trạng thái không hợp lệ (ví dụ từ resolved về open): Support ném INVALID_TRANSITION (409), transaction rollback nên không có audit log.
- when Ghi audit log thất bại trong transaction: Lỗi nổi lên từ callback nên transaction rollback và trạng thái ticket không đổi.
- when Không tìm thấy ticket theo id: Support ném TICKET_NOT_FOUND (404).

## Endpoints covered

- POST /support/tickets [module:support] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/support.md
- GET /support/tickets/me [module:support] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/support.md
- GET /admin/support/tickets [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/support/tickets/:id [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
