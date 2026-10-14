# Techniques used by admin

id: techniques:admin
module: module:admin

### Chặn tự khoá quyền (other:self-lockout-guard)

- how: Chặn tự ban (CANNOT_BAN_SELF), tắt manage_permissions của admin (CANNOT_DISABLE_PERMISSION_CONTROL), tự đổi role (CANNOT_CHANGE_OWN_ROLE) và hạ admin cuối (CANNOT_DEMOTE_LAST_ADMIN).
- why: Chặn tự ban (CANNOT_BAN_SELF), tắt manage_permissions của admin (CANNOT_DISABLE_PERMISSION_CONTROL), tự đổi role (CANNOT_CHANGE_OWN_ROLE) và hạ admin cuối (CANNOT_DEMOTE_LAST_ADMIN).
- applied here: Chặn tự ban (CANNOT_BAN_SELF), tắt manage_permissions của admin (CANNOT_DISABLE_PERMISSION_CONTROL), tự đổi role (CANNOT_CHANGE_OWN_ROLE) và hạ admin cuối (CANNOT_DEMOTE_LAST_ADMIN).
- endpoints: e4, e29, e31
- alternative: 

### RBAC có ma trận quyền chỉnh được lúc chạy (rbac-permission-matrix)

- how: Vai trò ánh xạ tới tập quyền chi tiết lưu trong DB; guard kiểm quyền cụ thể cho từng thao tác.
- why: Đổi quyền của một vai trò không cần deploy lại.
- applied here: Route admin dùng @RequireAdminPermission(AdminPermission.*) với bảng AdminRolePermission trong DB, quyền chỉnh được qua PATCH /admin/permissions/:role/:permission.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- alternative: Hard-code quyền theo vai trò: nhanh nhưng mỗi thay đổi phải sửa mã.

### RBAC theo vai trò (@Roles + RolesGuard) (rbac-role-guard)

- how: Vai trò (role) nằm trong token/hồ sơ; @Roles(...) + RolesGuard cho phép hoặc chặn theo vai trò.
- why: Đơn giản, dễ kiểm toán khi số vai trò ít và ổn định (user/admin).
- applied here: Controller admin bị giới hạn theo vai trò staff qua RolesGuard toàn cục và các endpoint quản lý role (/admin/staff/:id/role).
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- alternative: ABAC hoặc policy engine linh hoạt hơn nhưng phức tạp khi chỉ có vài vai trò.

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: Broadcast thông báo lọc người nhận theo thuộc tính VIP (activeVipUserIds, BroadcastAudience.Vip/All) trong admin.service.
- endpoints: e27
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Ban/unban, report, video, ticket, gift, config, broadcast, đổi role mở this.dataSource.transaction cùng auditLogService.record; hoàn tiền (e23) do ledger.reverse mở transaction riêng, audit qua callback withinTransaction.
- endpoints: e4, e5, e7, e8, e11, e12, e13, e17, e19, e20, e25, e26, e27, e29, e31
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: Video chuyển trạng thái bằng UPDATE có điều kiện status=from trong ShortVideoService.transitionOrThrow; ticket kiểm canTransition dưới pessimistic_write; sai trạng thái nguồn trả lỗi 409.
- endpoints: e11, e12, e13, e17
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: AdminService mở transaction rồi truyền manager vào supportService.setStatusWithManager và các service khác để chung một transaction.
- endpoints: e17, e27, e31
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: setStaffRole gọi UserService.lockRoleAssignmentContext khoá pessimistic_write mọi user role admin và target trong transaction để hai lần hạ quyền song song không xoá hết admin (CANNOT_DEMOTE_LAST_ADMIN).
- endpoints: e31
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Refund thủ công dùng key tất định admin-refund:{transactionId} nên bấm refund lại cùng giao dịch không sinh bút toán đảo thứ hai; double-reverse còn bị TRANSACTION_ALREADY_REVERSED chặn.
- endpoints: e23
- alternative: 

### Sửa sai bằng bút toán đảo (reversal), không sửa lịch sử (reversal-compensation)

- how: Sửa sai bằng bút toán đảo mới thay vì sửa hoặc xoá lịch sử.
- why: Lịch sử luôn đầy đủ và truy vết được.
- applied here: adminRefundTransaction không sửa giao dịch gốc: ledger.reverse ghi bút toán đảo chiều trỏ reversalOf và đánh dấu gốc Reversed bằng UPDATE có điều kiện status Completed.
- endpoints: e23
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: closePartyRoom ghi audit trong hook onTransition cùng transaction đóng phòng; livekit.deleteRoom và publish PartyRoomClosed chạy sau commit trong closeRoomById (e14); push broadcast (e27) cũng sau commit.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: ThrottlerGuard toàn cục áp giới hạn mặc định cho mọi route /admin/*.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: AdminPermissionGuard ném PERMISSION_FORBIDDEN khi route thiếu metadata @RequireAdminPermission hoặc không có request.user, nên route admin mới quên khai quyền bị deny mặc định.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: broadcastNotification đã commit in-app trước, push qua notificationService.sendPush là best-effort, lỗi không rollback broadcast.
- endpoints: e27
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Mỗi route admin có @ApiOperation và @ApiOkResponse với DTO riêng (ví dụ AdminDashboardDto) cho hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: Các danh sách phòng, video, giao dịch nhận CursorPageQueryDto (query.limit, query.cursor) và trả CursorPage.
- endpoints: e15, e22, e9, e10
- alternative: 

### Audit log hành động nhạy cảm (audit-log)

- how: Ghi audit log cho hành động nhạy cảm của admin.
- why: Truy trách nhiệm: mọi hành động nhạy cảm của admin để lại dấu vết gắn với người thực hiện.
- applied here: AdminService inject AuditLogService và ghi audit cho các hành động nhạy cảm như ban/unban, duyệt report, video, hoàn tiền.
- endpoints: e4, e5, e7, e8, e11, e12, e13, e14, e17, e19, e20, e23, e25, e26, e27, e29, e31, e1
- alternative: 

