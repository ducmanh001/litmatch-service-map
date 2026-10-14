# Techniques used by matching

id: techniques:matching
module: module:matching

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Các endpoint invite và ticket của matching đi qua JwtAuthGuard là APP_GUARD toàn cục; user lấy từ AuthenticatedUser.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Token thiết bị cho khách (HMAC) (guest-device-token)

- how: Token thiết bị cấp cho khách, ký HMAC, gắn với thiết bị để tính hạn mức.
- why: Khách tạo tài khoản vô hạn nên hạn mức phải bám thiết bị chứ không bám tài khoản.
- applied here: GuestMatchQuotaService inject GuestDeviceTokenService để xác thực token thiết bị (header X-Guest-Device-Token) cho guest khi vào hàng đợi.
- endpoints: 
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: MatcherWorkerService inject MATCH_INTERACTION_POLICY (Safety bind qua useExisting) làm cổng canPair khi ghép (e15) và accept invite (e4). Lúc ghép còn kiểm lại user status Active và gender 2 chiều; KHÔNG kiểm lại isGuest ở tryPair.
- endpoints: e7, e8, e9, e10, e11, e12
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Matcher gọi interactionPolicy.canPair(ta.userId, tb.userId): block ĐANG active, hoặc report/block trong SAFETY_REMATCH_COOLDOWN_DAYS thì không được ghép; cặp hợp lệ nhưng bị từ chối được trả lại queue với score cũ.
- endpoints: e15, e2, e4, e1
- alternative: 

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: MatchingService.assertOwnership so ticket.userId với user.userId, ném TICKET_FORBIDDEN 'Ticket không thuộc về bạn' cho các thao tác trên ticket.
- endpoints: e9, e10, e11, e12
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Giá speedup theo tier VIP do getSpeedupPriceDiamondForUser đọc server-side và là nguồn giá duy nhất cho debit lẫn TicketDto; client không gửi hay hard-code giá.
- endpoints: e8, e12
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: MatchTicket.idempotencyKey (length 255, unique) khử trùng vào hàng đợi; ticket-sweeper requeue dùng requeueIdempotencyKey(session.id, ticket.id) để tạo key xác định.
- endpoints: e16, e7, e12, e4
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Matcher ghép cặp trong dataSource.transaction(async (manager) => ...) và TicketSweeper expire mỗi session trong một transaction riêng.
- endpoints: e15, e16, e7, e10, e11, e5, e6, e4
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Matcher khoá MatchTicket bằng lock pessimistic_write theo thứ tự id ASC chống deadlock; sweeper khoá MatchSession rồi mới tới tickets.
- endpoints: e15, e16, e11, e10, e7, e4, e5, e6
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: tryPair gọi notificationService.createWithManager(manager, ...) cùng transaction ghép; joinQueue gọi economy.spendDiamondInTransaction(manager, ...) và guestQuota.consumeForMatch(manager, ...) cùng transaction tạo ticket.
- endpoints: e15, e7, e11, e10, e4, e5, e6
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: MatchTicket.idempotencyKey unique là chốt chặn cuối; MatchingService dùng isUniqueViolation/violatedConstraint để bắt vi phạm và xử lý thành lỗi nghiệp vụ.
- endpoints: e7, e1, e4
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: Invite chuyển trạng thái qua MATCH_INVITE_TRANSITIONS và assertTransition (pending->accepted/declined/cancelled/expired); chuyển trái luật ném INVITE_INVALID_TRANSITION.
- endpoints: e4, e5, e6
- alternative: 

### Hàng đợi chia shard (loại match / region) (sharded-queue)

- how: Hàng đợi chia shard theo loại match hoặc region.
- why: Nhiều matcher chạy song song không dẫm chân và giảm điểm nóng.
- applied here: Hàng đợi chia shard bằng matchingShardKey(matchType, region, ageBand); matcher lặp queue.listActiveShards() và drainShard từng shard.
- endpoints: e15, e16, e7
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: Đẩy lại Redis và publish realtime match.confirmed chỉ làm sau khi transaction DB commit; confirm lặp (replay) không bắn lại sự kiện.
- endpoints: e15, e11, e13
- alternative: 

### Tín hiệu trong cùng process (wake-up / EventEmitter) (in-process-signal)

- how: Tín hiệu trong cùng process (EventEmitter hoặc wake-up) đánh thức worker.
- why: Giảm độ trễ mà không cần broker khi chỉ một process quan tâm.
- applied here: MatcherWakeup là tập listener trong process; MatcherWorkerService subscribe để đánh thức vòng ghép khi có enqueue, kèm timer gom batch.
- endpoints: e15, e7
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: match.confirmed chỉ best-effort nên GET /matching/tickets/:id là đường poll chắc chắn; web refetch mỗi MATCHING_TICKET_REFETCH_INTERVAL_MS = 10 giây khi ticket còn queued/matched.
- endpoints: e9, e13
- alternative: 

### Lua script nguyên tử trong Redis (redis-lua-atomic)

- how: Script Lua chạy nguyên tử trong Redis: nhiều lệnh không bị xen giữa.
- why: Rate limit và lease cần đọc-so sánh-ghi không race.
- applied here: RedisRateLimitAdapter.consume chạy Lua INCR + EXPIRE + reservation trong một lệnh eval nên giới hạn mời mỗi giờ atomic giữa nhiều pod, có refund theo reservation.
- endpoints: e1
- alternative: 

### Hàng đợi ưu tiên Redis sorted set (ZADD/ZPOPMIN) (redis-zset-queue)

- how: ZADD đặt vé kèm điểm, ZPOPMIN lấy vé có điểm thấp nhất một cách nguyên tử.
- why: Hợp ghép đôi: nhiều worker không lấy trùng một vé và thứ tự theo điểm.
- applied here: RedisMatchingQueue (redis-matching-queue.adapter.ts) đẩy ticket vào sorted set theo shard bằng redis.zadd(shard, mode NX/XX, score, ticketId), score là thứ tự ưu tiên.
- endpoints: 
- alternative: Redis list không có điểm ưu tiên; Streams thừa khi trạng thái thật nằm ở PostgreSQL.

### Sweeper dọn dữ liệu hết hạn (sweeper)

- how: Job quét dọn bản ghi hết hạn.
- why: Giải phóng tài nguyên và tránh ghi rác.
- applied here: InviteSweeperService và TicketSweeperService quét định kỳ qua SchedulerRegistry: dọn invite Pending quá hạn và expire session/ticket quá hạn confirm.
- endpoints: e16, e14
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: InviteSweeperService và MatcherWorkerService dùng ManagedInterval (this.job.start) để chạy vòng lặp theo MATCHING_*_INTERVAL_MS khi bootstrap.
- endpoints: e14
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: InviteSweeperService và TicketSweeperService đều khởi động ManagedInterval với clusterSingleton: { dataSource }; pg_try_advisory_xact_lock giữ trong suốt task nên chỉ một replica chạy mỗi sweeper.
- endpoints: e14
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: assertNotExpired chặn accept/decline/cancel khi quá expiresAt nhưng chỉ set status trong bộ nhớ rồi throw nên transaction rollback, trạng thái Expired KHÔNG được lưu; InviteSweeper mới ghi expired và giải phóng uq_match_invites_pending_pair.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: POST /matching/invites và POST /matching/tickets gắn @Throttle({ default: { limit: 10, ttl: minutes(1) } }) riêng, chặt hơn mặc định của ThrottlerGuard toàn cục.
- endpoints: e1, e7, e2, e3, e4, e5, e6, e8, e9, e10, e11, e12
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: Publish realtime match.confirmed và ghi metric (observeMatched) sau commit là best-effort; lỗi không chặn luồng vì client còn poll GET /matching/tickets/:id.
- endpoints: e15, e11, e13
- alternative: 

### Rate limit bằng Redis (Lua) (rate-limit-redis)

- how: Bộ đếm dùng chung trong Redis (Lua) giới hạn theo khoá nghiệp vụ.
- why: Giới hạn đúng khi có nhiều instance.
- applied here: matchingRateLimitProvider cấp MATCHING_RATE_LIMIT bằng RedisRateLimitAdapter(redis) qua RateLimitPort, dùng giới hạn trên Redis cho vào hàng đợi và mời.
- endpoints: e7, e1
- alternative: 

### Hạn mức theo ngày (daily-quota)

- how: Hạn mức theo ngày lưu ở DB.
- why: Giới hạn lượt dùng miễn phí mỗi ngày để chặn lạm dụng mà vẫn cho thử tính năng; hạn mức lưu ở DB nên bền qua restart.
- applied here: Hạn mức miễn phí theo ngày: bảng match_daily_quotas (userId, ngày UTC, loại match) cho user đăng ký và guest_match_quotas (quotaDate) cho guest; hết quota thì trả Diamond.
- endpoints: e7
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Invite và matching controller khai báo @ApiOperation, @ApiCreatedResponse({ type: TicketDto }), @ApiHeader cho X-Guest-Device-Token để sinh OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /matching/invites nhận CursorPageQueryDto và trả MatchInvitesPageDto { items, nextCursor } theo phân trang cursor.
- endpoints: e2
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: Payload notification MatchConfirmed và event match.confirmed chỉ có sessionId/ticketId/matchType, không bao giờ chứa partnerId vì Soul Match ẩn danh tới khi unlock.
- endpoints: e11, e13
- alternative: 

### Băm HMAC có khoá (hmac-hashing)

- how: Băm HMAC với khoá bí mật.
- why: Không dò ngược được bằng bảng băm.
- applied here: GuestMatchQuotaService.keyHash dùng createHmac('sha256', MATCHING_GUEST_QUOTA_PEPPER) băm định danh guest để chống farm quota mà không lưu giá trị thô.
- endpoints: e7
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: assertInviteeAvailable trả cùng INVITE_TARGET_UNAVAILABLE 422 cho tự mời, id không tồn tại và invitee trong hidden-set nên không lộ ai đã block hoặc report mình.
- endpoints: e1
- alternative: 

