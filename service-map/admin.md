# Service map: Admin

id: service-map:admin
module: module:admin
deployable: core-api
confidence: high

API back-office: RBAC, audit, điều phối module

AdminModule là lớp điều phối back-office cho apps/admin: kiểm quyền (role + permission theo bảng admin_role_permissions), gọi đúng service của module sở hữu dữ liệu và ghi admin_audit_logs CÙNG transaction với hành động. Nó chỉ sở hữu ma trận quyền; KHÔNG sở hữu user/ví/ledger/video/phòng/report/ticket/gift/notification và không tự ghi vào bảng của họ.

Owns: bảng admin_role_permissions (role x permission -> enabled); điều phối ghi bảng admin_audit_logs qua common/audit (bảng thuộc AuditModule, append-only bằng DB trigger)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/admin/dashboard | admin | Dashboard tổng hợp: thống kê user, doanh số diamond 7 ngày, số phòng đang mở và 10 hoạt động audit gần nhất. Cần permission view_users. |
| e2 | rest | GET | /api/v1/admin/users | admin | Danh sách user lọc theo status/role/nickname, phân trang offset. Cần view_users. |
| e3 | rest | GET | /api/v1/admin/users/:id | admin | Xem hồ sơ nội bộ của một user (status + role). Cần view_users. |
| e4 | rest | POST | /api/v1/admin/users/:id/ban | admin | Khoá tài khoản user và ghi audit user.banned trong cùng transaction; chặn tự ban chính mình (CANNOT_BAN_SELF). Cần ban_users. |
| e5 | rest | POST | /api/v1/admin/users/:id/unban | admin | Mở khoá tài khoản user và ghi audit user.unbanned cùng transaction. Cần ban_users. |
| e6 | rest | GET | /api/v1/admin/reports | admin | Danh sách báo cáo vi phạm (report) lọc và phân trang offset. Cần resolve_reports. |
| e7 | rest | POST | /api/v1/admin/reports/:id/resolve | admin | Đánh dấu report là resolved + audit report.resolved cùng transaction. Cần resolve_reports. |
| e8 | rest | POST | /api/v1/admin/reports/:id/dismiss | admin | Bác bỏ report (dismissed) + audit cùng transaction. Cần resolve_reports. |
| e9 | rest | GET | /api/v1/admin/videos/pending | admin | Danh sách video chờ duyệt (chế độ VIDEO_MODERATION_MODE=pre). Cần resolve_reports. |
| e10 | rest | GET | /api/v1/admin/videos/published | admin | Danh sách video đang published (mới nhất trước) để admin gỡ thủ công. Cần resolve_reports. |
| e11 | rest | POST | /api/v1/admin/videos/:id/approve | admin | Duyệt video pending_review -> published + audit video.approved cùng transaction. Cần resolve_reports. |
| e12 | rest | POST | /api/v1/admin/videos/:id/reject | admin | Từ chối video pending_review -> rejected + audit video.rejected cùng transaction. Cần resolve_reports. |
| e13 | rest | POST | /api/v1/admin/videos/:id/remove | admin | Gỡ thủ công video published -> removed + audit video.removed, bổ sung cho auto-hide theo ngưỡng report. Cần resolve_reports. |
| e14 | rest | POST | /api/v1/admin/rooms/:id/close | admin | Đóng cưỡng bức party room (reason AdminClosed) với audit trong cùng transaction đóng phòng; dọn LiveKit/realtime chạy sau commit trong party-room. Cần manage_rooms. |
| e15 | rest | GET | /api/v1/admin/rooms | admin | Danh sách party room kèm số member, phân trang cursor. Cần manage_rooms. |
| e16 | rest | GET | /api/v1/admin/support/tickets | admin | Danh sách ticket hỗ trợ của mọi user, lọc status, phân trang cursor. Cần manage_support. |
| e17 | rest | PATCH | /api/v1/admin/support/tickets/:id | admin | Đổi trạng thái ticket (kiểm canTransition) và staffResponse + audit support.ticket.updated cùng transaction. Cần manage_support. |
| e18 | rest | GET | /api/v1/admin/gifts | admin | Xem toàn bộ catalog quà kể cả quà đã tắt. Cần manage_gifts. |
| e19 | rest | POST | /api/v1/admin/gifts | admin | Tạo quà mới (code unique uq_gifts_code) + audit gift.created cùng transaction. Cần manage_gifts. |
| e20 | rest | PATCH | /api/v1/admin/gifts/:id | admin | Sửa giá/tên/thứ tự/bật-tắt quà (không hard-delete) + audit gift.updated cùng transaction. Cần manage_gifts. |
| e21 | rest | GET | /api/v1/admin/economy/wallet/:userId | admin | Xem ví (balance snapshot + VIP) của một user, chỉ đọc. Cần refund_transaction. |
| e22 | rest | GET | /api/v1/admin/economy/users/:userId/transactions | admin | Lịch sử giao dịch của user, phân trang cursor, actor-scoped. Cần refund_transaction. |
| e23 | rest | POST | /api/v1/admin/economy/transactions/:id/refund | admin | Hoàn tiền thủ công một giao dịch bằng bút toán đảo (ledger.reverse), bắt buộc có reason, audit cùng transaction; không sửa/xoá giao dịch gốc. Cần refund_transaction. |
| e24 | rest | GET | /api/v1/admin/config/economy-catalog | admin | Xem catalog kinh tế: gói IAP diamond và gói VIP kèm trạng thái active. Cần manage_config. |
| e25 | rest | PATCH | /api/v1/admin/config/iap-products/:productId | admin | Bật/tắt một IAP product + audit economy.iap_product.updated cùng transaction. Cần manage_config. |
| e26 | rest | PATCH | /api/v1/admin/config/vip-plans/:id | admin | Bật/tắt một gói VIP + audit economy.vip_plan.updated cùng transaction. Cần manage_config. |
| e27 | rest | POST | /api/v1/admin/notifications/broadcast | admin | Gửi thông báo broadcast tới nhóm all/vip/non-vip: lưu in-app + audit trong một transaction, rồi push best-effort sau commit. Cần manage_config. |
| e28 | rest | GET | /api/v1/admin/permissions | admin | Đọc ma trận quyền role (moderator/admin) x permission từ admin_role_permissions. Cần manage_permissions. |
| e29 | rest | PATCH | /api/v1/admin/permissions/:role/:permission | admin | Bật/tắt một permission cho role admin/moderator + audit; không cho tắt manage_permissions của admin (tránh tự khoá). Cần manage_permissions. |
| e30 | rest | GET | /api/v1/admin/staff | admin | Danh sách nhân sự (user có role admin/moderator). Cần manage_permissions. |
| e31 | rest | PATCH | /api/v1/admin/staff/:id/role | admin | Đổi role nhân sự với khoá pessimistic để không hạ admin cuối cùng; chặn tự đổi role chính mình; audit cùng transaction. Cần manage_permissions. |

## Coordination edges (why this service talks to others)

### ví, hoàn tiền, catalog, VIP (c1) with economy [service, out, di-call]

- endpoints: e1, e21, e22, e23, e24, e25, e26, e27
- what: EconomyService: getWallet, listTransactions, adminRefundTransaction(transactionId, reason, actorUserId, withinTransaction), listCatalogForAdmin, setIapProductActive/setVipPlanActive(manager), listActiveVipUserIds, getAdminAnalytics.
- why: Economy là writer duy nhất của LedgerEntry/Wallet (double-entry append-only, balance cập nhật cùng transaction). Admin không được sửa ví hay xoá bút toán: hoàn tiền phải là reversal qua ledger.reverse với idempotency key tất định admin-refund:{id} và callback audit chạy chung DB transaction.
- if missing: Admin phải ghi thẳng bảng wallet/ledger: lệch sổ cái, refund đúp, mất dấu vết actor+reason. Catalog IAP/VIP và analytics cũng phải đọc bảng của Economy qua đường vòng.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:313, apps/core-api/src/modules/admin/admin.service.ts:326, apps/core-api/src/modules/admin/admin.service.ts:344
- review: sound — Đã đối chiếu ledger.reverse: key admin-refund:{id}, callback withinTransaction chạy cùng manager, UPDATE có điều kiện status chặn double-reverse.

### tạo/sửa Gift catalog (c2) with gift [service, out, di-call]

- endpoints: e18, e19, e20
- what: GiftService.listAllForAdmin, createGift(manager, input), updateGift(manager, giftId, input); manager là EntityManager của transaction do AdminService mở.
- why: Bảng gifts thuộc module gift (code unique uq_gifts_code, gift_events tham chiếu FK nên cấm xoá cứng). Truyền cùng manager để việc đổi catalog và dòng audit commit/rollback cùng nhau.
- if missing: Admin tự ghi bảng gifts: bỏ qua kiểm tra trùng code và quy tắc 'tắt thay vì xoá', hoặc catalog đổi mà không có audit (hoặc ngược lại).
- evidence: apps/core-api/src/modules/admin/admin.service.ts:271, apps/core-api/src/modules/admin/admin.service.ts:276, apps/core-api/src/modules/admin/admin.service.ts:297
- review: sound — createGift bắt unique violation uq_gifts_code (migration party-room-gift), updateGift không hard-delete; cùng manager với audit.

### broadcast + push (c3) with notification [service, out, di-call]

- endpoints: e27
- what: NotificationService.createBroadcastWithManager(manager, userIds, {title, body}) tạo Notification type AdminBroadcast (chunk 500), rồi sendPush(notification) cho từng bản ghi sau commit; sendPush gọi provider di động rồi web-push và nuốt mọi lỗi (notification.service.ts:93-118).
- why: Notification là chủ bảng notifications và các kênh push (FCM/APNs/web-push). Ghi in-app trong CÙNG transaction với audit để broadcast không 'gửi 1 nửa'; push là best-effort sau commit (sendPush không bao giờ throw) nên lỗi provider không rollback broadcast đã ghi.
- if missing: Admin phải tự ghi notification và gọi provider push: lệch với quy tắc best-effort (push lỗi làm hỏng broadcast) và phá ranh giới sở hữu của notification.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:438, apps/core-api/src/modules/admin/admin.service.ts:463
- review: sound — createBroadcastWithManager chunk 500 cùng manager; sendPush bọc try/catch nuốt lỗi; gọi sau transaction commit.

### đóng phòng, đếm phòng (c4) with party-room [service, out, di-call]

- endpoints: e1, e14, e15
- what: PartyRoomService.closeRoomById(roomId, AdminClosed, undefined, onTransition) với hook ghi audit; listRoomsWithMemberCounts; countActiveRooms cho dashboard.
- why: Đóng phòng là transition có khoá pessimistic_write trên row room và idempotent; chỉ party-room biết dọn SFU (livekit.deleteRoom) và publish PartyRoomClosed sau commit. Audit được nhét vào hook onTransition để cùng transaction với việc đổi status phòng.
- if missing: Admin đổi status phòng trực tiếp: LiveKit room vẫn chạy, member không nhận PartyRoomClosed, và có race với host-leave/webhook/sweeper đóng cùng lúc.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:214, apps/core-api/src/modules/admin/admin.service.ts:234, apps/core-api/src/modules/admin/admin.service.ts:637
- review: sound — closeRoomById khoá pessimistic_write, trả closed:false khi đã đóng, hook onTransition trong transaction; livekit.deleteRoom và publish sau commit.

### đọc/đổi trạng thái report (c5) with safety [service, out, di-call]

- endpoints: e6, e7, e8
- what: SafetyService.findReportsPage(filter, limit, offset) và setReportStatus(manager, reportId, status) với status resolved hoặc dismissed.
- why: Report thuộc module safety (trung lập, module khác như short-video ghi report qua nó). Chỉ status được phép đổi, các field khác bất biến; truyền manager để đổi status và audit report.{status} cùng commit.
- if missing: Admin sửa bảng reports tùy ý: có thể đổi nội dung báo cáo gốc hoặc đóng report mà không có dấu vết audit.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:112, apps/core-api/src/modules/admin/admin.service.ts:129
- review: sound — safety.setReportStatus chỉ gán report.status, cùng manager với audit report.{status}; lưu ý không có kiểm chuyển trạng thái hay khoá.

### duyệt/gỡ video ngắn (c6) with short-video [service, out, di-call]

- endpoints: e9, e10, e11, e12, e13
- what: ShortVideoService.listPendingReview, listPublished({sort:'recent'}), adminApprove/adminReject/adminRemove(videoId, manager) thực hiện transition pending_review->published|rejected, published->removed.
- why: Vòng đời video (status machine, auto-hide theo ngưỡng report, VIDEO_MODERATION_MODE) thuộc short-video; transitionOrThrow chỉ cho phép chuyển từ đúng trạng thái nguồn. Admin đi qua đó và chung manager để audit và status commit cùng nhau.
- if missing: Admin ghi status video thẳng: phê duyệt/gỡ sai trạng thái nguồn (vd duyệt video đã removed) và không đồng bộ với logic auto-hide.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:151, apps/core-api/src/modules/admin/admin.service.ts:157, apps/core-api/src/modules/admin/admin.service.ts:162
- review: sound — transitionOrThrow dùng UPDATE có điều kiện status=from (compare-and-set), ném VIDEO_INVALID_TRANSITION nếu sai nguồn.

### danh sách/đổi status ticket (c7) with support [service, out, di-call]

- endpoints: e16, e17
- what: SupportService.listAll(limit, cursor, status) và setStatusWithManager(manager, ticketId, {status, staffResponse}).
- why: Support giữ state machine ticket (canTransition, khoá pessimistic_write) và là chủ bảng ticket; admin chỉ đi qua đó, dùng chung manager để đổi trạng thái và audit support.ticket.updated atomic.
- if missing: Admin đổi status ticket bất kỳ (vd closed -> open) bỏ qua INVALID_TRANSITION, hoặc ticket đổi mà không có audit.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:242, apps/core-api/src/modules/admin/admin.service.ts:251
- review: sound — support.service dùng pessimistic_write và canTransition (line 83, 92); admin chỉ gọi setStatusWithManager.

### user, ban, role nhân sự (c8) with user [service, out, di-call]

- endpoints: e1, e2, e3, e4, e5, e27, e30, e31
- what: UserService: getByIdOrThrow, findPage, banUser/unbanUser(manager), listActiveUserIds, listStaff, lockRoleAssignmentContext + setRoleWithManager, getAdminStats, findByIds. Guard AdminPermissionGuard cũng gọi getByIdOrThrow để đọc role hiện tại mỗi request.
- why: Bảng users và status/role thuộc module user. Đổi role cần khoá pessimistic_write toàn bộ admin + target trong cùng transaction để kiểm adminCount, tránh 2 admin hạ nhau đồng thời làm hệ thống không còn admin (CANNOT_DEMOTE_LAST_ADMIN). Ban + audit atomic.
- if missing: Admin sửa bảng users thẳng: race hạ quyền admin cuối cùng và ban không có audit. Nếu AdminPermissionGuard không đọc lại role từ DB (admin.service.ts:473) thì demote chỉ có hiệu lực khi access token cũ hết hạn, vì RolesGuard chỉ nhìn role trong JWT.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:53, apps/core-api/src/modules/admin/admin.service.ts:61, apps/core-api/src/modules/admin/admin.service.ts:77
- review: sound — lockRoleAssignmentContext khoá pessimistic_write mọi admin và target; AdminPermissionGuard gọi hasPermission rồi getByIdOrThrow mỗi request, RolesGuard chỉ đọc req.user.role từ JWT.

### audit log + ma trận quyền (c9) with postgres [infra, out, shared-db]

- endpoints: e1, e4, e5, e7, e8, e11, e12, e13, e14, e17, e19, e20, e23, e25, e26, e27, e28, e29, e31
- what: AuditLogService.record({actorUserId, action, targetType, targetId, metadata}, manager) ghi admin_audit_logs (16 call site) và listRecent(10) cho dashboard; repo AdminRolePermission đọc/ghi admin_role_permissions.
- why: Hành động nhạy cảm không được thành công một nửa (docs/06): dòng audit phải nằm trong cùng DB transaction với thay đổi nghiệp vụ nên cần chung manager; audit chỉ INSERT, DB trigger chặn UPDATE/DELETE để không sửa được lịch sử. Ma trận quyền nằm DB để chỉnh quyền không cần deploy.
- if missing: Hành động admin không truy vết được, hoặc audit ghi ngoài transaction nên có thể có hành động mà thiếu log (hoặc ngược lại); đổi quyền phải sửa code và deploy.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:78, apps/core-api/src/modules/admin/admin.service.ts:94, apps/core-api/src/modules/admin/admin.service.ts:134
- review: sound — Đếm được đúng 16 lời gọi auditLogService.record trong admin.service; trigger trg_admin_audit_logs_append_only có trong migration 1753700000000.

### REST + Bearer token admin (c50) with admin-console [client, in, http]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- what: apps/admin (Vite/React) gọi toàn bộ /api/v1/admin/* bằng apiClient sinh từ OpenAPI; mọi request kèm Authorization: Bearer, refresh token là cookie httpOnly + csrfToken lưu localStorage.
- why: Admin là SPA riêng, không có session server: quyền được quyết định hoàn toàn ở core-api (role + permission) nên client chỉ hiển thị. Phải gửi Bearer access token của tài khoản staff và xử lý 403 vì ma trận quyền đổi được lúc runtime.
- if missing: Không có console thao tác; hoặc client tin quyền ở phía UI và route nào quên guard sẽ bị lộ (hiện core-api deny route thiếu permission).
- evidence: libs/api-client/src/lib/client.ts:70, apps/admin/src/features/economy/api.ts:58
- review: sound — Client gắn Bearer, csrfToken lưu localStorage (token-store), controller deny route thiếu permission (AdminPermissionGuard).

### outbox event refund (c51) with kafka [infra, out, kafka]

- endpoints: e23
- what: Refund thủ công đi qua ledger.reverse với outboxEventTypeOverride 'economy.diamond.refunded'; Economy ghi OutboxEvent cùng transaction bút toán đảo, relay publish lên Kafka sau commit.
- why: Event outbox ghi CÙNG transaction với bút toán đảo (docs/03 § 3.6) nên không mất hoặc phát đôi khi commit lỗi; eventType riêng economy.diamond.refunded (outboxEventTypeOverride, truyền xuống applyWalletDeltas ở ledger.service.ts) tách refund khỏi ghi có/ghi nợ thường. Relay Kafka mặc định tắt (kể cả hosted) và không có consumer của topic trong apps/, nên hiện outbox_events chỉ tích lũy.
- if missing: Khi relay được bật, downstream không phân biệt được refund với ghi có thường hoặc không nhận được refund; hiện (relay tắt) chưa có tác động runtime.
- evidence: apps/core-api/src/modules/economy/economy.service.ts:756
- review: sound — Relay tắt mặc định và cả trong deploy/production, k8s, hosted; grep không thấy consumer topic litmatch.economy.events trong apps/.

## Notes

- Không có doc riêng cho admin trong docs/services; controller trích docs/12 § 12.7 và docs/10. Code là nguồn chính.
- Comment đầu AdminController còn ghi 'Task 0 ... endpoint tối thiểu' nhưng controller hiện có 31 route đủ các nhóm users/reports/videos/rooms/support/gifts/economy/config/permissions/staff.
- Chuỗi guard thực tế: ThrottlerGuard, JwtAuthGuard, RolesGuard (APP_GUARD, app.module.ts) rồi @RequireRoles(Admin, Moderator) ở class và AdminPermissionGuard (@UseGuards ở class). Route nào thiếu @RequireAdminPermission bị deny (403), không mặc định mở.
- JwtAuthGuard KHÔNG đọc DB và KHÔNG kiểm status banned mỗi request (chỉ verify chữ ký + TTL). AdminPermissionGuard đọc lại role từ DB nên hạ role có hiệu lực ngay, nhưng hasPermission không kiểm user.status: staff đã bị ban vẫn dùng được API admin tới khi access token hết hạn. Nghi vấn lỗ hổng cần xác nhận.
- Quyền xem ví và lịch sử giao dịch (e21, e22) dùng permission refund_transaction (không có permission 'view economy' riêng); video ngắn dùng permission resolve_reports (nhãn 'Duyệt báo cáo & video ngắn').
- resolve/dismiss report chỉ đổi Report.status, KHÔNG có state machine (Safety không chặn resolved->dismissed) và KHÔNG tự ban user/gỡ nội dung; ban và gỡ video là call riêng. Khác với support dùng canTransition.
- e22 lịch sử giao dịch là actor-scoped (comment trong admin.service.ts): chưa thấy giao dịch user chỉ là người nhận quà; refund thủ công vì vậy chỉ tìm được từ phía người trả.
- e27 broadcast đọc TOÀN BỘ id user active (listActiveUserIds) vào bộ nhớ, insert theo chunk 500 trong 1 transaction rồi sendPush Promise.all không giới hạn song song: rủi ro scale khi user nhiều. Danh sách recipient được đọc TRƯỚC transaction nên user mới/bị ban giữa chừng có thể lệch.
- apps/admin còn gọi GET /api/v1/party/rooms (route không thuộc /admin) trong features/rooms/api.ts: client admin không chỉ dùng prefix /admin.
- Cửa sổ staff bị ban: hasPermission (admin.service.ts:469-481) chỉ kiểm role và policy, không kiểm user.status; banUser không thu hồi refresh session và JwtAuthGuard không đọc DB, nên staff bị ban dùng được API admin tới khi access token hết hạn (mặc định JWT_ACCESS_TTL_SECONDS=900s), sau đó refresh bị chặn. Suy ra từ code, chưa chạy thử.
- unbanUser (admin.service.ts:88-) không có kiểm tự-unban tương ứng CANNOT_BAN_SELF của ban. Kết hợp cửa sổ ở trên, staff vừa bị ban nhưng còn ban_users và token hợp lệ có thể tự mở khoá trong cửa sổ đó. Suy ra từ code, chưa chạy thử.
- Outbox economy.diamond.refunded do refund thủ công ghi chỉ có ý nghĩa khi relay bật: ECONOMY_OUTBOX_RELAY_ENABLED mặc định false (env.validation.ts:349) và hosted cũng false (deploy/hosted/core-api.env.example:37); grep apps/ không thấy consumer của topic economy ngoài chính module economy.
