# Techniques used by movie-match

id: techniques:movie-match
module: module:movie-match

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Tạo phiên xem chung chỉ cho bạn bè: `friendService.areFriends(userId, friendUserId)` sai thì ném `MovieMatchErrors.NOT_FRIEND`.
- endpoints: e1, e5
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Server tự chọn video anon từ MOVIE_MATCH_ANON_VIDEO_URLS, client không gửi URL nên người lạ không ép video tuỳ ý.
- endpoints: e5
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ chat ẩn danh (e11) bắt Idempotency-Key: cột idempotencyKey unique với prefix movieMessageIdempotencyKey; cùng key khác nội dung trả 409 MESSAGE_IDEMPOTENCY_CONFLICT; tạo phiên bạn bè idempotent theo cặp.
- endpoints: e11
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Tạo phiên và các bước ghép ẩn danh chạy trong `this.dataSource.transaction(async (manager) => ...)` để ghi session và trạng thái cùng một đơn vị công việc.
- endpoints: e1, e3, e5, e6, e7, e8, e9, e10
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: Flow ẩn danh truyền `manager: EntityManager` vào helper như `lockAnonMatcher(manager)` để lock và đọc/ghi cùng transaction.
- endpoints: e5, e7, e6, e8, e9, e10
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Khi ghép/kết thúc ẩn danh, `manager.findOne(MovieSession, { lock: { mode: 'pessimistic_write' } })` khoá dòng session đang active.
- endpoints: e7, e5, e6, e8, e9, e10
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Chống race tạo phiên: bắt `isUniqueViolation(err)` rồi đọc lại `findActiveByPair`; cột `idempotencyKey` có `unique: true` chặn tin nhắn trùng.
- endpoints: e1, e5, e11
- alternative: 

### Advisory lock (pg_advisory) để serialize / singleton (advisory-lock)

- how: pg_advisory_xact_lock khoá theo khoá logic, không cần có hàng cụ thể, giữ đến hết transaction.
- why: Tuần tự hoá việc không gắn với một hàng (ghép đôi, worker singleton).
- applied here: `lockAnonMatcher` chạy `pg_advisory_xact_lock(hashtext($1))` với `MOVIE_MATCH_ADVISORY_LOCK_KEY` để tuần tự hoá việc ghép đôi ẩn danh.
- endpoints: e5, e7
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: Entity chat ẩn danh trong movie-match-anon.entities.ts được khai báo APPEND-ONLY: chỉ thêm tin, sống theo session, không sửa.
- endpoints: 
- alternative: 

### Redis pub/sub fanout (at-most-once, không replay) (redis-pubsub-fanout)

- how: core-api PUBLISH lên kênh realtime:user:{id}; gateway PSUBSCRIBE rồi đẩy xuống socket; không lưu, không replay.
- why: Sự kiện chỉ là gợi ý còn REST là nguồn sự thật nên at-most-once là đủ và rất rẻ.
- applied here: Bốn event movie.* được đóng gói `RealtimeEnvelope` rồi `publishRealtimeEvent(this.redis, ...)` tới từng participant qua Redis pub/sub; client lỡ thì đọc lại bằng REST.
- endpoints: e14, e15, e16, e17
- alternative: Redis Streams hoặc Kafka bền và replay được nhưng cần consumer group + ack: thừa với tin gợi ý.

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: Web poll GET /anon/current và anon messages mỗi 5s (MOVIE_ANON_*_REFETCH_INTERVAL_MS), phiên bạn bè 10s; luồng ẩn danh không phát realtime khi ghép/rating/chat nên chỉ poll thấy.
- endpoints: e4, e6, e12
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: Event started/ended chỉ được publish sau khi transaction tạo hoặc kết thúc phiên đã commit, trong `Promise.all` trên hai userId.
- endpoints: e14, e16
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: lazyExpireAnon đặt watchEndedAt = expiresAt và isPlaying=false khi request tới sau hạn, mở phase rating mà không terminal; không có @Cron/worker.
- endpoints: e5, e6, e8, e9, e10
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: Phiên ẩn danh có cột `expiresAt`; hết hạn được xử lý lười qua `expireAnonForUser(manager, userId)` khi `getAnonCurrent` được gọi, không có worker.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Reaction ẩn danh giới hạn riêng `@Throttle({ default: { limit: 60, ttl: minutes(1) } })` ngoài ThrottlerGuard toàn cục.
- endpoints: e13, e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: publishRealtimeEvent bọc try/catch, lỗi Redis chỉ log warn; nghiệp vụ movie.* vẫn thành công và client dùng REST poll.
- endpoints: e2, e8, e9, e13, e14, e15, e16, e17
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Controller mô tả từng route bằng `@ApiOperation` (vd POST sessions idempotent) để sinh hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: `GET anon/sessions/:id/messages` nhận `@Query() query: CursorPageQueryDto` và trả `MovieMessagesPageDto` phân trang theo cursor.
- endpoints: e12
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getSessionForParticipant/lockedAnonSession/anonSessionForParticipant gộp không tồn tại, sai mode và không phải participant thành cùng 404 NOT_FOUND (chống IDOR).
- endpoints: e2, e3, e4, e8, e9, e10, e11, e12, e13
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: Reaction chỉ gồm sessionId + emoji whitelist, không persist và không kèm userId; anonView trả vai trò tương đối nên không lộ partner trước khi mutual-like.
- endpoints: e13, e17
- alternative: 

