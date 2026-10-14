# Techniques used by palm-match

id: techniques:palm-match
module: module:palm-match

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint /palm-match/* đi qua JwtAuthGuard là APP_GUARD toàn cục; reading vẫn dùng được cho guest.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: lockedActiveSession yêu cầu dòng PalmMatchActiveParticipant(userId từ JWT, sessionId) rồi khoá session pessimistic_write; không khớp trả 404 SESSION_NOT_FOUND chung.
- endpoints: e4, e5
- alternative: 

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Khi ghép, PalmMatchService gọi safetyService.canPair(userId, candidate.user_id) kiểm lại block/report ngay lúc chọn cặp, không tin state lúc enqueue.
- endpoints: e2
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Seed hash(userId, category, ngày UTC) tính hoàn toàn ở server nên client không chọn/quay lại kết quả khác trong cùng ngày; template chọn qua ORDER BY id để deterministic.
- endpoints: e1
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: rate() truyền manager vào FriendService.ensureFriendship để Friendship+Conversation cùng transaction với kết quả phiên; helper private lockMatcher/expireActiveForUser dùng chung manager của joinQueue/getCurrent.
- endpoints: e2, e6, e3, e4, e5
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: joinQueue, getCurrent, flip, rate, dismissCurrent bọc dataSource.transaction; chỉ joinQueue/dismissCurrent lấy lockMatcher, getCurrent/joinQueue gọi expireActiveForUser, flip/rate khoá session qua lockedActiveSession.
- endpoints: e2, e3, e4, e5, e6
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Khi huỷ/kết thúc, manager.findOne(PalmMatchSession, { lock: { mode: 'pessimistic_write' } }) khoá session trước khi completeSession(Cancelled).
- endpoints: e6, e2, e3, e4, e5
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Advisory lock (pg_advisory) để serialize / singleton (advisory-lock)

- how: pg_advisory_xact_lock khoá theo khoá logic, không cần có hàng cụ thể, giữ đến hết transaction.
- why: Tuần tự hoá việc không gắn với một hàng (ghép đôi, worker singleton).
- applied here: lockMatcher chạy pg_advisory_xact_lock(hashtext(PALM_MATCH_ADVISORY_LOCK_KEY)) để tuần tự hoá việc ghép cặp toàn hệ thống trong transaction.
- endpoints: e2, e6
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Vào queue bằng insert PalmMatchQueueEntry với orIgnore(); khoá chính của queue/active-participant là chốt DB cuối chống trùng khi vào hàng đợi lặp.
- endpoints: e2
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: Palm Match không phát realtime; web useCurrentPalmMatch refetchInterval PALM_MATCH_REFETCH_INTERVAL_MS (5s) khi state queued|active là kênh duy nhất biết đã ghép.
- endpoints: e3
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: getCurrent/joinQueue/flip/rate gọi expireActiveForUser/expireSession: session Active quá expiresAt chuyển Expired lúc có request, queue quá PALM_MATCH_QUEUE_MAX_WAIT_SECONDS bị xoá lúc đọc; không có worker.
- endpoints: e2, e3, e4, e5
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: getCurrent gọi expireActiveForUser để lazy-expire deadline phía server khi user reload/reconnect, không cần sweeper riêng.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Các endpoint palm-match chịu ThrottlerGuard là APP_GUARD toàn cục trong app.module.ts, dùng hạn mức mặc định.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: PalmMatchController khai báo @ApiOperation cho reading, queue, current, flip, rating để sinh OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

