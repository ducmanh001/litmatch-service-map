# Service map: Support

id: service-map:support
module: module:support
deployable: core-api
confidence: high

Phiếu hỗ trợ và vòng đời xử lý

Sở hữu bảng support_tickets: người dùng gửi phản hồi/yêu cầu hỗ trợ (idempotent) và xem lại phiếu của mình; vòng đời trạng thái open -> in_progress -> resolved -> closed. Không có endpoint cho staff: staff xử lý qua module admin gọi vào SupportService, và support không gửi thông báo hay realtime khi staff trả lời.

Owns: bảng support_tickets (userId, category, message, status, staffResponse, idempotencyKey); unique uq_support_ticket_user_idempotency (userId, idempotencyKey)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/support/tickets | jwt | Gửi phiếu hỗ trợ (category + message tối đa 2000 ký tự) ở trạng thái open; bắt buộc header Idempotency-Key, cùng key gửi lại thì trả phiếu cũ, khác nội dung thì 409. |
| e2 | rest | GET | /api/v1/support/tickets/me | jwt | Liệt kê phiếu của chính mình theo cursor (createdAt, id) giảm dần để theo dõi trạng thái và staffResponse. |

## Coordination edges (why this service talks to others)

### phiếu + unique idempotency (c1) with postgres [infra, out, shared-db]

- endpoints: e1, e2
- what: Ghi/đọc bảng support_tickets: save phiếu mới, findOneByOrFail(userId, idempotencyKey) khi replay, query builder phân trang theo cursor, và findOne khoá pessimistic_write khi đổi status.
- why: Idempotency được giữ bởi unique index (userId, idempotencyKey) trong DB: khi client retry mạng yếu hoặc bấm đúp, request thứ hai vi phạm unique và được đọc lại thành replay thay vì tạo phiếu trùng; check-then-insert ở tầng ứng dụng sẽ race.
- if missing: Retry/bấm đúp tạo nhiều phiếu giống nhau; hai staff đổi status đồng thời có thể ghi đè nhau.
- evidence: apps/core-api/src/modules/support/entities/support-ticket.entity.ts:18
- review: sound

### staff cập nhật phiếu (c2) with admin [service, in, di-call]

- endpoints: 
- what: AdminService.listSupportTickets gọi SupportService.listAll(limit, cursor, status); updateSupportTicket mở transaction và gọi setStatusWithManager(manager, ticketId, {status, staffResponse}).
- why: Admin truyền EntityManager để đổi status/staffResponse và ghi audit log 'support.ticket.updated' cùng một transaction (hành động staff nhạy cảm không được thành công một nửa). Support khoá hàng phiếu rồi kiểm canTransition để hai staff song song không nhảy trạng thái sai thứ tự.
- if missing: Đổi status mà không có audit (hoặc ngược lại), hoặc phiếu bị chuyển từ closed về trạng thái cũ.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:242, apps/core-api/src/modules/admin/admin.service.ts:251
- review: sound

### Idempotency-Key, đọc phiếu (c50) with web [client, in, http]

- endpoints: e1, e2
- what: apps/web gửi POST /support/tickets với header Idempotency-Key do client sinh (idempotencyKey nằm trong input của mutation) và đọc danh sách phiếu qua GET /support/tickets/me?limit=20; sau khi tạo phiếu mutation invalidate query danh sách.
- why: Server chỉ chống trùng nếu client dùng lại đúng một key cho cùng lần gửi (unique userId+idempotencyKey). Không có push/realtime khi staff trả lời và web không có polling (không refetchInterval trong features/support), nên staffResponse mới chỉ hiện khi danh sách được đọc lại.
- if missing: Thiếu header thì e1 bị từ chối; client sinh key mới mỗi lần retry thì mất chống trùng; không đọc lại danh sách thì user không thấy phản hồi của staff.
- evidence: apps/web/src/features/support/api.ts:36
- review: sound

## Notes

- Không có docs/services cho support; mô tả dựa trên code.
- Module chỉ có 2 route cho user (POST /support/tickets, GET /support/tickets/me). Staff list/update nằm ở module admin (listAll, setStatusWithManager), nên admin console không gọi support trực tiếp.
- Không có thông báo/realtime khi staff đổi status hoặc trả lời (staffResponse): không có event, job hay lời gọi NotificationService ở support/admin. Web cũng không poll: features/support/api.ts chỉ có useQuery đọc danh sách và useMutation tạo phiếu rồi invalidate supportKeys.mine (api.ts:41-42), không có refetchInterval; phản hồi staff chỉ hiện khi query được đọc lại (mở màn hình hoặc sau khi gửi phiếu).
- Máy trạng thái: canTransition cho phép Open -> InProgress/Resolved/Closed, InProgress -> Resolved/Closed, Resolved -> Closed; Closed là cuối; giữ nguyên status cũng hợp lệ.
