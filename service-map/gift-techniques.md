# Techniques used by gift

id: techniques:gift
module: module:gift

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Cả 4 endpoint gift đi qua APP_GUARD JwtAuthGuard toàn cục trong app.module.ts; không route nào gắn @Public nên đều cần bearer token.
- endpoints: e1, e2, e3, e4
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: Khi tặng quà, GiftService đọc receiver.isGuest: người nhận là guest thì pointsAwarded = 0, còn lại tính Math.floor(priceDiamond * GIFT_POINTS_RATE_PERCENT / 100).
- endpoints: e2, e3, e4
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Client chỉ gửi giftId; giá lấy từ bảng gifts (active=true) lúc tặng, receiver video suy từ video.authorUserId, pointsAwarded tính phía server theo GIFT_POINTS_RATE_PERCENT.
- endpoints: e2, e3, e4
- alternative: 

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Tặng quà profile gọi profileSocial.assertNotBlocked hai lần: trước khi tra gift và lặp lại trong withinTransaction, để block phát sinh giữa chừng làm rollback ledger thay vì thành đường bypass Safety.
- endpoints: e4
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: giftSendIdempotencyKey = gift:send:{senderUserId}:{clientKey} gửi xuống Economy; chỉ e2, e3, e4 yêu cầu header Idempotency-Key, replay trả lại GiftEvent cũ qua transactionId unique.
- endpoints: e2, e3, e4
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: Economy gọi withinTransaction(manager, transactionId) để Gift ghi GiftEvent và Notification cùng transaction ledger (e2-e4); admin createGift/updateGift nhận manager do AdminService mở.
- endpoints: e2, e3, e4
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Trong withinTransaction của quà profile, Gift gọi lockProfileForChatInManager khoá hàng users của profile để first-contact mở bằng quà tuần tự với đường chat miễn phí.
- endpoints: e4
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Entity có unique ở DB: Gift.code (uq_gifts_code, bắt race hai admin tạo trùng) và GiftEvent.transactionId unique để mỗi giao dịch chỉ ghi một sự kiện quà.
- endpoints: 
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: publishGiftSent chỉ chạy sau khi economy.sendGift đã commit và khi !result.replayed, đọc lại getActiveRoomMembers để fanout gift.sent cho member đang ở phòng lúc đó.
- endpoints: e5
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Cả ba route tặng quà (phòng, video, profile) gắn @Throttle({ default: { limit: 30, ttl: minutes(1) } }) trên nền ThrottlerGuard toàn cục; GET /gifts chỉ dùng throttle mặc định.
- endpoints: e2, e3, e4, e1
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: publishGiftSent là best-effort: lỗi fanout gift.sent không làm hỏng giao dịch tặng quà đã commit vì client còn đọc lại bằng REST.
- endpoints: e5
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: GiftController mô tả bằng @ApiOperation/@ApiOkResponse({ type: [GiftDto] }) cho catalog và các endpoint tặng quà, làm nguồn cho OpenAPI.
- endpoints: e1, e2, e3, e4
- alternative: 

