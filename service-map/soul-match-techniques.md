# Techniques used by soul-match

id: techniques:soul-match
module: module:soul-match

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint /soul-match/sessions/:id/* đi qua JwtAuthGuard là APP_GUARD toàn cục.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: SoulMatchService xác định partner của session và gọi friendService.areFriends(user.userId, partnerId) để trả cờ matched khi rating.
- endpoints: e4, e6, e1, e2, e3
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: Gửi message soul chat kiểm tra profile.status === UserStatus.Banned và ném SoulMatchErrors.USER_BANNED 'Tài khoản bị khoá'.
- endpoints: e3
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ POST messages (e3) dùng header Idempotency-Key: SoulChatMessage.idempotencyKey prefix soul:msg:{userId}:{key}, unique uq_soul_chat_messages_idempotency_key; rating (e4) idempotent nhờ unique (sessionId, raterUserId).
- endpoints: e3, e4
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: Phase chatting|rating|closed không lưu entity mà derive trong getRoomForMember từ confirmedAt + SOUL_CHAT_DURATION_SECONDS/SOUL_RATING_WINDOW_SECONDS; matched (areFriends) giữ phòng ở chatting vô hạn.
- endpoints: e1, e2, e3, e4, e6
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Unique (sessionId, raterUserId) của soul_match_ratings và unique idempotency key của message; service bắt isUniqueViolation rồi trả bản ghi đã có.
- endpoints: e3, e4
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Gửi message chạy trong dataSource.transaction(async (manager) => ...) để khoá session, kiểm tra trạng thái rồi lưu message.
- endpoints: e3, e4
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Gửi message và rating cùng khoá dòng MatchSession bằng lock pessimistic_write, để message không lọt giữa bước snapshot và chuyển sang Friend Chat.
- endpoints: e3, e4
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: soul_chat_messages và soul_match_ratings chỉ INSERT (không updatedAt, không API xoá/sửa); rating immutable, phòng đóng chỉ khoá qua API để giữ bằng chứng cho T&S.
- endpoints: e3, e4
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: SoulChatMessage và SoulMatchRating là append-only (không updatedAt, không extend BaseAppEntity); rating bất biến để chống thăm dò đổi ý.
- endpoints: 
- alternative: 

### Redis pub/sub fanout (at-most-once, không replay) (redis-pubsub-fanout)

- how: core-api PUBLISH lên kênh realtime:user:{id}; gateway PSUBSCRIBE rồi đẩy xuống socket; không lưu, không replay.
- why: Sự kiện chỉ là gợi ý còn REST là nguồn sự thật nên at-most-once là đủ và rất rẻ.
- applied here: publishRealtimeEvent đẩy envelope soul.* lên realtime:user:{userId} của cả hai thành viên qua client SOUL_MATCH_REDIS; gateway relay xuống socket.
- endpoints: e7, e8, e9
- alternative: Redis Streams hoặc Kafka bền và replay được nhưng cần consumer group + ack: thừa với tin gợi ý.

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: soul.message và soul.matched chỉ publish sau khi dataSource.transaction trả về và chỉ cho bản ghi mới (newlyMatched / không phải replay unique-violation).
- endpoints: e7, e8
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: Web useSoulMessages refetchInterval SOUL_MESSAGES_REFETCH_INTERVAL_MS (5s) và session 10s làm fallback thật khi socket soul.* rớt.
- endpoints: e1, e2
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: POST .../messages gắn @Throttle({ default: { limit: 60, ttl: minutes(1) } }) riêng cho chat ngắn, trên nền ThrottlerGuard toàn cục.
- endpoints: e3, e1, e2, e4, e5, e6
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: Realtime soul.message được publish sau khi persist, chỉ cho message mới, best-effort; senderRole tính riêng cho từng người nhận.
- endpoints: e3, e4, e7, e8, e9
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: SoulMatchController khai báo @ApiOperation với summary từng endpoint (vd trạng thái phòng chat ẩn danh) để sinh OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /soul-match/sessions/:id/messages nhận CursorPageQueryDto và trả SoulMessagesPageDto theo phân trang cursor.
- endpoints: e2
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getRoomForMember gộp session không tồn tại và caller không phải thành viên thành cùng 404 SESSION_NOT_FOUND để không làm oracle dò sessionId.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: senderRole me|partner được tính riêng từng người nhận ngay trong core-api nên payload soul.message và danh sách message không lộ userId đối phương trước khi match.
- endpoints: e2, e7
- alternative: 

