# Techniques used by notification

id: techniques:notification
module: module:notification

### Dọn subscription chết (other:prune-dead-subscription)

- how: Adapter web-push trả removeSubscription khi push service báo 404/410; NotificationService.sendPush gọi removeById để xoá hàng web_push_subscriptions chết khỏi bảng.
- why: Adapter web-push trả removeSubscription khi push service báo 404/410; NotificationService.sendPush gọi removeById để xoá hàng web_push_subscriptions chết khỏi bảng.
- applied here: Adapter web-push trả removeSubscription khi push service báo 404/410; NotificationService.sendPush gọi removeById để xoá hàng web_push_subscriptions chết khỏi bảng.
- endpoints: e3, e4
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4, e5
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: `markRead` tải notification rồi kiểm `notification.userId !== userId` thì ném `NOTIFICATION_NOT_FOUND`, user khác không đánh dấu được.
- endpoints: e5
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Index `uq_web_push_subscriptions_endpoint` unique trên `endpoint` làm chốt chặn; `webPushSubscriptions.upsert(user.userId, dto)` đăng ký lại không nhân đôi.
- endpoints: e3
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: `createWithManager(manager: EntityManager, input)` cho module khác (feed) tạo Notification trong cùng transaction nghiệp vụ.
- endpoints: 
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: Notification không có realtime event riêng; web poll GET /notifications/unread-count mỗi UNREAD_NOTIFICATION_COUNT_REFETCH_INTERVAL_MS = 60_000 để cập nhật badge chuông.
- endpoints: e2
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4, e5
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: `sendPush` best-effort: gọi sau khi Notification commit và không bao giờ throw, push lỗi không phá luồng gốc.
- endpoints: 
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: `@ApiOperation`, `@ApiCursorPageQuery()`, `@ApiOkResponse({ type: NotificationsPageDto })` mô tả hợp đồng OpenAPI của từng route.
- endpoints: e1, e2, e3, e4, e5
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: `GET /notifications` nhận `CursorPageQueryDto` và trả `NotificationsPageDto` có `nextCursor`.
- endpoints: e1
- alternative: 

### Cột JSONB (jsonb-columns)

- how: Cột JSONB lưu dữ liệu linh hoạt.
- why: Cấu trúc thay đổi thường xuyên.
- applied here: Cột `payload` của `Notification` kiểu `jsonb` lưu dữ liệu tuỳ theo `type`.
- endpoints: 
- alternative: 

### Cô lập secret (chỉ core-api giữ khoá) (secret-isolation)

- how: Khoá và secret chỉ nằm ở core-api, không ra client.
- why: Giảm bề mặt lộ khoá: khoá ký và secret chỉ nằm ở backend, phía client không bao giờ nhận được.
- applied here: endpoint web-push của trình duyệt được coi là credential, không log ra ngoài adapter; khoá riêng VAPID chỉ nằm ở core-api, sw.js không chứa auth/token.
- endpoints: e3, e4
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: payload notification chỉ chứa dữ liệu tối thiểu để client tự render, không hardcode text; match_confirmed chỉ có sessionId/matchType, không partnerId (Soul Match ẩn danh).
- endpoints: e1
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: markRead trả NOTIFICATION_NOT_FOUND 404 cho cả notification không tồn tại lẫn của user khác, nên không lộ sự tồn tại của notificationId người khác.
- endpoints: e5
- alternative: 

