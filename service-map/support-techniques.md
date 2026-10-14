# Techniques used by support

id: techniques:support
module: module:support

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: JwtAuthGuard toàn cục bảo vệ /support/tickets, người gửi lấy từ @CurrentUser().
- endpoints: e1, e2
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: GET /support/tickets/me gọi listMine(user.userId) và SupportService.list lọc ticket.userId = userId, nên người dùng chỉ thấy phiếu của mình; listAll chỉ admin gọi.
- endpoints: e2
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: POST /support/tickets (e1) bắt buộc Idempotency-Key; unique (userId, idempotencyKey) cho replay trả phiếu cũ, khác category/message thì 409 IDEMPOTENCY_CONFLICT. GET /support/tickets/me (e2) không dùng key.
- endpoints: e1
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Index unique uq_support_ticket_user_idempotency là chốt chặn cuối; SupportService dùng isUniqueViolation/violatedConstraint để xử lý replay.
- endpoints: e1
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: Phiếu luôn khởi tạo ở SupportTicketStatus.Open; các bước tiếp theo (open, in_progress, resolved, closed) chỉ đổi qua setStatusWithManager với canTransition, closed là trạng thái cuối.
- endpoints: e1
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: SupportService.setStatusWithManager nhận EntityManager để Admin cập nhật ticket trong cùng transaction với audit log.
- endpoints: 
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: setStatusWithManager đọc ticket với lock pessimistic_write trước khi đổi status, tránh hai staff ghi đè nhau.
- endpoints: 
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: ThrottlerGuard toàn cục áp rate limit mặc định cho các route support.
- endpoints: e1, e2
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: SupportController gắn @ApiOperation và @ApiCreatedResponse({ type: SupportTicketDto }) cho hợp đồng OpenAPI.
- endpoints: e1, e2
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /support/tickets/me nhận CursorPageQueryDto (ListSupportTicketsQueryDto) và trả SupportTicketsPageDto kèm CursorPageMeta.
- endpoints: e2
- alternative: 

