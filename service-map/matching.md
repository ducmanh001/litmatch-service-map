# Service map: Matching

id: service-map:matching
module: module:matching
deployable: core-api
confidence: high

Hàng đợi ghép cặp, ticket, invite, MatchSession

Sở hữu vòng đời MatchTicket (queued→confirmed|expired|cancelled), hàng đợi Redis sharded + matcher worker, MatchInvite (mời có chủ đích) và MatchSession là bản ghi duy nhất của một cặp đã ghép; là nơi DUY NHẤT ghi match_sessions. KHÔNG sở hữu ví/ledger (Economy), block/report (Safety), hồ sơ (User), chat Soul (Soul Match) hay call LiveKit (Calling).

Owns: bảng match_tickets (partial unique uq_match_tickets_active_user: 1 user 1 ticket queued/matched); bảng match_sessions (nguồn sự thật của cặp đã ghép, Soul/Voice); bảng match_invites (partial unique pending theo cặp); bảng match_daily_quotas + guest_match_quotas (quota miễn phí theo ngày UTC, guest theo HMAC user/device/IP); Redis ZSET matching:queue:{matchType}:{region}:{ageBand} + SET matching:shards:active; Redis rate-limit key lời mời mỗi giờ

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/matching/invites | jwt | Tạo lời mời Voice/Soul tới invitee: chặn tự mời, rate limit Redis theo giờ, từ chối kín (oracle-safe) nếu invitee nằm trong hidden-set, lưu MatchInvite pending (unique pending theo cặp) rồi tạo notification + push. |
| e2 | rest | GET | /api/v1/matching/invites | jwt | Inbox lời mời ĐANG pending gửi tới mình (cursor); re-check hidden-set Safety mỗi lần đọc và batch-load profile người mời qua UserService. |
| e3 | rest | GET | /api/v1/matching/invites/:id | jwt | Chi tiết một lời mời, chỉ inviter hoặc invitee xem được (người khác 403) kèm inviterProfile. |
| e4 | rest | POST | /api/v1/matching/invites/:id/accept | jwt | Invitee chấp nhận: transaction riêng precheck lazy-expire + canPair (fail thì invite thành declined), rồi 1 transaction tạo 2 MatchTicket confirmed + MatchSession confirmed, idempotent theo inviteId, publish match.confirmed sau commit. |
| e5 | rest | POST | /api/v1/matching/invites/:id/decline | jwt | Invitee từ chối lời mời pending (chỉ invitee, lazy-expire theo expiresAt tại thời điểm hành động). |
| e6 | rest | POST | /api/v1/matching/invites/:id/cancel | jwt | Inviter huỷ lời mời đã gửi trước khi có phản hồi (chỉ inviter). |
| e7 | rest | POST | /api/v1/matching/tickets | jwt | Vào hàng đợi: Throttle 10 req/phút, bắt buộc Idempotency-Key, 1 transaction khoá user + trừ quota ngày (guest cần X-Guest-Device-Token) + trừ DIA nếu useDiamond + tạo ticket queued; sau commit mới ZADD vào Redis. 409 nếu đã có ticket active. |
| e8 | rest | GET | /api/v1/matching/tickets/current | jwt | Phục hồi UI sau reload: trả ticket queued/matched của chính user (hoặc null) kèm giá speedup theo tier VIP. |
| e9 | rest | GET | /api/v1/matching/tickets/:id | jwt | Poll trạng thái ticket, chỉ chủ ticket xem được (chống IDOR); đây là fallback khi socket match.confirmed bị lỡ. |
| e10 | rest | DELETE | /api/v1/matching/tickets/:id | jwt | Huỷ ticket của chính mình, chỉ khi còn queued; đổi trạng thái trong DB rồi ZREM khỏi shard Redis. |
| e11 | rest | POST | /api/v1/matching/tickets/:id/confirm | jwt | Đường tương thích cho session pending_confirm cũ: khoá session trước ticket, đủ 2 bên confirm thì chuyển cả hai ticket + session sang confirmed trong 1 transaction rồi publish match.confirmed + push. UI mới không gọi. |
| e12 | rest | POST | /api/v1/matching/tickets/:id/speedup | jwt | Mua ưu tiên hàng đợi: chỉ khi ticket queued, trừ DIA qua Economy (key speedup:{user}:{key}, replay không trừ lại), cộng priorityBoostMs trong DB rồi set score Redis tuyệt đối (ZADD XX); không giới hạn số lần. |
| e13 | socket-emit | EMIT | match.confirmed | internal | Báo match.confirmed {ticketId, sessionId} cho từng người trong cặp qua Redis realtime:user:{userId} sau commit ghép; best-effort, client vẫn có GET /matching/tickets/:id để poll. |
| e14 | job | WORKER | InviteSweeperService.onApplicationBootstrap | internal | Job dọn lời mời pending quá MATCHING_INVITE_TTL_SECONDS sang expired (housekeeping, không phải chốt correctness vì accept/decline/cancel tự lazy-expire); chạy 1 replica nhờ advisory lock Postgres. |
| e15 | job | WORKER | MatcherWorkerService.onApplicationBootstrap | internal | Matcher worker: được đánh thức khi có enqueue (debounce) hoặc theo interval backstop, quét matching:shards:active, ZPOPMIN 2 ticket mỗi shard, verify trong 1 transaction FOR UPDATE (status, gender 2 chiều, canPair) rồi tạo MatchSession confirmed + notificatio… |
| e16 | job | WORKER | TicketSweeperService.onApplicationBootstrap | internal | Sweeper: UPDATE có điều kiện queued→expired cho ticket chờ quá MATCHING_QUEUE_MAX_WAIT_SECONDS rồi ZREM khỏi Redis; dọn cả session pending_confirm legacy quá hạn. |

## Coordination edges (why this service talks to others)

### xác thực device token guest (c1) with auth [service, out, di-call]

- endpoints: e7
- what: GuestMatchQuotaService.authorize gọi GuestDeviceTokenService.verifyForUser(token, userId) với header X-Guest-Device-Token để lấy device key (JWT HS256 aud litmatch-matching, ràng buộc sub=userId), rồi băm HMAC (user, device, IP) làm khoá quota guest. Phía client hiện thiếu hợp đồng: Auth phát guestDeviceToken ở /auth/guest nhưng apps/web không lưu/gửi header này (xem notes).
- why: Guest tạo được tài khoản vô hạn nên quota phải bám vào device chứ không chỉ userId; chỉ Auth giữ secret ký token (AUTH_GUEST_DEVICE_TOKEN_SECRET) và ràng buộc token với userId, Matching không được tự tin deviceId thô từ client.
- if missing: Guest bịa deviceId/đăng ký guest mới liên tục để farm lượt match miễn phí, quota guest chống farm (MATCHING_GUEST_DAILY_LIMIT) mất tác dụng.
- evidence: apps/core-api/src/modules/matching/services/guest-match-quota.service.ts:67
- review: sound — Đã đối chiếu: token HS256 aud litmatch-matching, khoá quota băm HMAC (user, device, IP); web không gửi X-Guest-Device-Token (không có kết quả grep).

### trừ DIA + tra tier VIP (c2) with economy [service, out, di-call]

- endpoints: e7, e8, e9, e10, e11, e12
- what: spendDiamondInTransaction(manager, ..., MatchingExtraMatch) khi mua lượt thêm lúc tạo ticket; spendDiamond(..., MatchingSpeedup) cho speedup; getActiveVipTier(userId) để chọn giá speedup regular/VIP/SVIP cho mọi TicketDto trả về.
- why: Tiền chỉ được đổi trong ledger của Economy (double-entry, idempotencyKey unique). Mua lượt thêm phải nằm cùng transaction tạo ticket (nên truyền EntityManager) để không có ticket miễn phí khi trừ tiền lỗi hoặc ngược lại; giá theo tier do server đọc ngay trước khi debit, client không hard-code.
- if missing: Match/speedup trả phí không trừ được DIA, hoặc trừ tiền mà ticket không được tạo (và retry trừ đôi), giá VIP/SVIP không áp dụng đúng.
- evidence: apps/core-api/src/modules/matching/matching.service.ts:139, apps/core-api/src/modules/matching/matching.service.ts:264, apps/core-api/src/modules/matching/matching.service.ts:448
- review: sound — Extra match trừ DIA cùng transaction tạo ticket (spendDiamondInTransaction); speedup gọi spendDiamond riêng, giá VIP/SVIP đọc ngay trước khi debit.

### tạo + đẩy thông báo match (c3) with notification [service, out, di-call]

- endpoints: e1, e11, e15
- what: createWithManager(manager, MatchConfirmed {sessionId, matchType}) ngay trong transaction ghép cặp, sau commit gọi sendPush(n); với invite: create(MatchInviteReceived) rồi sendPush cho invitee.
- why: Người đang thoát app không có socket nên push (gửi sau commit, best-effort) là kênh chủ động báo 'đã ghép xong'; notification row ghi bằng cùng manager nên atomic với session/ticket. Payload cố tình không chứa partnerId vì Soul Match ẩn danh tới khi unlock (comment matching.service.ts:377).
- if missing: Người dùng không ở trong app không biết đã ghép/được mời; nếu push chạy trong transaction thì lỗi provider làm rollback cả cuộc ghép.
- evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:299, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:387, apps/core-api/src/modules/matching/matching.service.ts:381, apps/core-api/src/modules/matching/services/invite.service.ts:153
- review: corrected — Overstated: push không phải kênh duy nhất (còn notification row và realtime socket); push thực ra gửi sau commit, best-effort.

### canPair + hidden-set (block) (c4) with safety [service, out, di-call]

- endpoints: e1, e2, e4, e15
- what: MatchInteractionPolicy.canPair(userA, userB) được bind useExisting: SafetyService; matcher gọi lúc ghép, accept invite gọi trong precheckAccept. Tạo invite/đọc inbox gọi getHiddenUserIds(userId) để loại người đã block/report.
- why: Block/report có thể phát sinh SAU khi ticket nằm chờ hoặc SAU khi mời, nên phải verify lại đúng lúc ghép/accept (docs/10 § 10.0.C); chỉ Safety giữ dữ liệu Report/Block append-only. canPair được gọi khi transaction ghép/precheckAccept đang mở (không truyền manager nên chạy trên kết nối riêng của Safety); accept thất bại canPair thì ghi invite thành declined ngay trong precheckAccept. Hidden-set còn dùng để trả lỗi kín (oracle-safe) khỏi lộ ai đã chặn mình.
- if missing: Hai người đã block/report nhau vẫn bị ghép hoặc chấp nhận invite; người bị chặn vẫn thấy lời mời trong inbox.
- evidence: apps/core-api/src/modules/matching/matching.module.ts:70, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:277, apps/core-api/src/modules/matching/services/invite.service.ts:475, apps/core-api/src/modules/matching/services/invite.service.ts:199
- review: sound — canPair chạy trên kết nối riêng của Safety khi transaction đang mở; precheckAccept ghi Declined rồi commit trước khi throw — đúng code.

### profile công khai người mời (c5) with user [service, out, di-call]

- endpoints: e1, e2, e3, e4, e5, e6
- what: InviteController gọi UserService.getByIdOrThrow(inviterUserId) và findByIds(...) (batch cho inbox) để ghép inviterProfile: PublicProfileDto vào MatchInviteDto.
- why: Invitee phải thấy tối thiểu nickname/avatar/gender trước khi quyết định (doc § 9.5); hồ sơ thuộc User nên Matching lấy qua public API, dùng findByIds để tránh N+1 và không rò birth date/region/trust score.
- if missing: Lời mời chỉ có userId trơ trọi, invitee không biết ai mời; hoặc Matching tự đọc bảng users và có nguy cơ lộ trường riêng tư.
- evidence: apps/core-api/src/modules/matching/controllers/invite.controller.ts:107, apps/core-api/src/modules/matching/controllers/invite.controller.ts:138, apps/core-api/src/modules/matching/controllers/invite.controller.ts:145
- review: sound — Đã kiểm PublicProfileDto.from chỉ gán id/nickname/gender/avatarId/interests; MatchInviteDto.from dùng nó cho inviterProfile.

### đọc/khoá hàng users (c6) with user [service, out, shared-db]

- endpoints: e4, e7, e15
- what: manager.findOneOrFail(User, lock pessimistic_write) chỉ ở bước join (guest authorize: is_guest, status, region, birthDate, trustScore); tryPair và accept invite chỉ manager.find(User) KHÔNG khoá, đọc status/gender trong cùng transaction.
- why: Join khoá hàng user để upgrade-khỏi-guest và join có thứ tự rõ; tryPair/accept đọc user tươi trong chính transaction ticket/session để verify ban/gender (không khoá user). UserService không có hàm đọc nhận EntityManager nên Matching đọc entity User trực tiếp, lệch doc § 9.5.
- if missing: Có thể ghép người vừa bị ban hoặc sai gender preference, và JWT guest cũ sau upgrade vẫn giữ quota guest.
- evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:246, apps/core-api/src/modules/matching/services/guest-match-quota.service.ts:53, apps/core-api/src/modules/matching/services/invite.service.ts:336
- review: corrected — pessimistic_write chỉ có ở GuestMatchQuotaService.authorize (join); tryPair và acceptInvite chỉ manager.find(User) không khoá.

### publish match.confirmed (c7) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e4, e11, e13, e15
- what: publishRealtimeEvent(realtime, logger, userId, {event:'match.confirmed', data:{ticketId, sessionId}}) cho từng người, từ matcher, accept invite và confirm legacy, lên Redis realtime:user:{userId} sau khi commit.
- why: Người đang chờ queue cần biết ngay đã được ghép để vào phòng, nhưng core-api không giữ socket; gateway relay envelope đã tính sẵn từng người. Publish phải SAU commit để client refetch thấy ticket đã confirmed, và best-effort vì đã có poll GET ticket.
- if missing: Client chỉ biết ghép xong ở lần poll REST kế tiếp (trễ vài giây); nếu publish trước commit, client refetch thấy ticket vẫn queued.
- evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:377, apps/core-api/src/modules/matching/matching.module.ts:86, apps/core-api/src/modules/matching/matching.service.ts:402
- review: sound — Publish nằm sau transaction ở matcher, acceptInvite và confirmTicket; publishRealtimeEvent nuốt lỗi và log warn.

### queue ZSET + rate limit (c8) with redis [infra, out, redis-state]

- endpoints: e1, e7, e10, e12, e15, e16
- what: Redis ZSET matching:queue:{matchType}:{region}:{ageBand} (ZADD NX/XX, ZPOPMIN 2, ZREM), SET matching:shards:active (SADD/SREM/SMEMBERS), và RateLimitPort.consume cho lời mời mỗi giờ.
- why: Hàng đợi chờ ghép cần lấy 2 ticket điểm thấp nhất nguyên tử giữa nhiều pod (ZPOPMIN, 2 matcher không bao giờ lấy trùng) mà không quét bảng; Postgres vẫn là nguồn sự thật nên sau pop phải verify lại. Rate limit lời mời cần bộ đếm INCR+EXPIRE nguyên tử.
- if missing: Matcher phải quét/khoá bảng match_tickets liên tục, hai pod có thể ghép trùng một ticket; không chặn được spam lời mời theo giờ.
- evidence: apps/core-api/src/modules/matching/redis/redis-matching-queue.adapter.ts:26, apps/core-api/src/modules/matching/redis/matching-redis.provider.ts:29, apps/core-api/src/modules/matching/matching.service.ts:625, apps/core-api/src/modules/matching/services/invite.service.ts:109
- review: sound — ZPOPMIN atomic (comment drainShard); rate limit là Lua INCR+EXPIRE kèm reservation nên đúng ý atomic.

### ticket/session/invite + khoá (c9) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e14, e15, e16
- what: Ghi/đọc match_tickets, match_sessions, match_invites, match_daily_quotas, guest_match_quotas; SELECT FOR UPDATE trên 2 ticket khi ghép; partial unique uq_match_tickets_active_user; advisory lock pg_try_advisory_xact_lock cho job chạy 1 replica.
- why: Các invariant phải chặn ở DB chứ không chỉ ở code: 1 user chỉ 1 ticket queued/matched (docs/06), Idempotency-Key unique để retry không tạo ticket/trừ quota đôi, và verify + queued→confirmed + tạo MatchSession phải cùng 1 transaction khoá hàng.
- if missing: Hai request song song tạo 2 ticket cho 1 user hoặc 2 pod cùng chốt một ticket vào hai session; retry mạng trừ quota/DIA đôi.
- evidence: apps/core-api/src/database/migrations/1752200000000-matching-core.ts:35, apps/core-api/src/modules/matching/entities/match-invite.entity.ts:44, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:236, apps/core-api/src/common/scheduling/managed-interval.ts:128
- review: sound — Partial unique uq_match_tickets_active_user, FOR UPDATE order id ASC, pg_try_advisory_xact_lock giữ suốt task trong ManagedInterval đều đúng.

### Calling đọc/đóng session (c10) with calling [service, in, di-call]

- endpoints: e4, e11, e15
- what: CallingService.joinCall gọi MatchingService.findSessionById để kiểm tra người gọi là thành viên, matchType=voice, status=confirmed; endCall gọi endVoiceSession (và endVoiceSessionForCall) để Matching chốt MatchSession ended cho cả cặp.
- why: MatchSession chỉ Matching được ghi (boundary docs/03); Calling chỉ điều phối LiveKit nên không được mint token cho session không phải voice/không còn confirmed, và khi call kết thúc phải đóng session bền vững để không join lại được.
- if missing: Calling mint token cho session lạ/đã kết thúc hoặc phải tự ghi match_sessions, trạng thái cặp lệch giữa hai module.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:105, apps/core-api/src/modules/calling/calling.service.ts:167, apps/core-api/src/modules/calling/calling.service.ts:317
- review: sound — joinCall kiểm member/voice/confirmed và re-check sau khi tạo CallSession; endVoiceSession(ForCall) tồn tại, ghi MatchSession trong Matching.

### Soul Match đọc/khoá session (c11) with soul-match [service, in, di-call]

- endpoints: e4, e11, e15
- what: SoulMatchService gọi findSessionById để dựng phòng, endSoulSession khi một bên rời, và manager.findOne(MatchSession, lock pessimistic_write) trong sendMessage/rating để khoá hàng match_sessions (chỉ khoá, không ghi).
- why: Phòng chat ẩn danh không có entity riêng mà là view dẫn xuất từ MatchSession do Matching tạo, và việc khoá hàng session là cách serialize hai rating 'like' song song; quyền ghi trạng thái ended vẫn giữ ở Matching.
- if missing: Soul Match phải giữ bản sao cặp/trạng thái hoặc ghi thẳng match_sessions, lệch với Matching và phá ownership.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:105, apps/core-api/src/modules/soul-match/soul-match.service.ts:163, apps/core-api/src/modules/soul-match/soul-match.service.ts:311
- review: sound — endSoulSession ghi trong Matching; Soul Match chỉ khoá pessimistic_write trong sendMessage/rating (comment nêu rõ serialize rating).

### Idempotency-Key + poll (c50) with web [client, in, http]

- endpoints: e7, e8, e9, e10, e12
- what: Web gửi POST /matching/tickets với header Idempotency-Key sinh theo intent, dùng GET /matching/tickets/current để phục hồi sau reload và poll GET /matching/tickets/:id (refetchInterval 10s khi còn queued/matched).
- why: Server trả match.confirmed chỉ best-effort nên REST poll là đường đảm bảo; Idempotency-Key giữ nguyên qua retry để join/speedup không tạo ticket hay trừ DIA lần hai.
- if missing: Mạng chập chờn làm tạo ticket/trừ tiền trùng; socket lỡ thì client treo ở màn chờ không biết đã ghép.
- evidence: apps/web/src/features/matching/api.ts:48, apps/web/src/features/matching/api.ts:86
- review: sound — MATCHING_TICKET_REFETCH_INTERVAL_MS = 10_000 và header Idempotency-Key đã xác nhận trong apps/web/src/features/matching/api.ts.

### match.confirmed làm refetch (c51) with web [client, out, socket]

- endpoints: e13
- what: Màn QueueStatusPanel lắng nghe RealtimeEvents.MatchConfirmed qua useRealtimeEvent; khi trùng ticketId thì invalidate query ticket (hoặc nhận ticket mới nếu đang rảnh) rồi GET ticket để chuyển vào phòng soul/voice.
- why: Event chỉ là gợi ý để refetch sớm, không chứa trạng thái đầy đủ; GET ticket mới là nguồn sự thật, nên client phải refetch thay vì tin payload. Invite accept cũng dùng chung event nên client phải nhận ticket mới khi đang rảnh.
- if missing: Người chờ phải đợi chu kỳ poll 10s mới vào phòng; người được mời accept xong không được kéo vào phòng.
- evidence: apps/web/src/features/matching/components/queue-status-panel.tsx:102
- review: sound — Event chỉ gợi ý invalidate, GET ticket là nguồn sự thật; khớp comment queue-status-panel.

## Notes

- Lệch doc/code: Swagger summary của POST /matching/tickets/:id/speedup ghi 'rate limit theo giờ, chặn TRƯỚC khi trừ tiền' nhưng code speedup() không có rate limit nào; doc matching-service.md § 4 nói đúng là KHÔNG giới hạn số lượt. Mô tả endpoint đã đi theo code/doc.
- Lệch doc/code: doc § 3 nói sweeper tính hạn 'kể từ createdAt', code ticket-sweeper.service.ts dùng enqueued_at (comment trong code nói rõ có chủ ý).
- Căng thẳng ranh giới: doc § 9.5 nhấn mạnh "không query repository User từ Matching" nhưng matcher tryPair, accept invite và guest-match-quota đọc/khoá thẳng entity User bằng manager.find/findOneOrFail trong transaction của Matching. Đã đối chiếu UserService: chỉ các hàm ghi/khoá chuyên biệt (createWithManager, lockRoleAssignmentContext, setRoleWithManager, adjustTrustScore, banUser, unbanUser) nhận EntityManager; getByIdOrThrow/findByIds/findActiveByIds thì không, nên Matching không có đường hợp lệ để đọc user tươi trong cùng transaction qua public API.
- Quan hệ với Safety bị AST hụt một phần: canPair đi qua token MATCH_INTERACTION_POLICY (useExisting: SafetyService trong matching.module.ts) nên dossier chỉ thấy getHiddenUserIds; đã bổ sung canPair vào c4 bằng evidence tay.
- Xác nhận bằng code: POST /matching/tickets với user guest (is_guest đọc TƯƠI từ DB, không tin JWT) bắt buộc header X-Guest-Device-Token, thiếu thì 401 MATCHING_GUEST_DEVICE_TOKEN_REQUIRED (guest-match-quota.service.ts:60-66). Auth có phát token này trong response /auth/guest (auth.controller.ts:171 guestDeviceToken) nhưng apps/web và apps/admin không có chỗ nào lưu hoặc gửi nó (grep guestDeviceToken|x-guest-device-token trong */src: 0 kết quả; login-form.tsx guestLogin chỉ gửi deviceId rồi bỏ qua phần còn lại của response). Hệ quả: guest đăng nhập từ web không vào được queue; chỉ user đã upgrade mới vào được. Chưa chạy thật để bắt lỗi 401 nhưng đường code không có nhánh nào khác.
- MatcherWakeup chỉ là event in-process (Set listener): enqueue trên pod A chỉ đánh thức matcher của pod A; pod khác dựa vào interval backstop MATCHING_MATCHER_INTERVAL_MS và job backstop chỉ chạy 1 replica nhờ advisory lock Postgres.
- Xác nhận bằng code (không chỉ suy luận): ZPOPMIN lấy ticket ra khỏi Redis TRƯỚC khi tryPair mở transaction Postgres (matcher-worker.service.ts:200 rồi :210). Chỉ nhánh "trả lại" mới ZADD NX lại (:207 khi pop được 1 ticket, :358-364 sau commit khi cặp không ghép được). Nếu tryPair ném lỗi (DB lỗi, SafetyService.canPair lỗi, tạo notification lỗi) thì exception nổi lên runOnce và hai ticket vừa pop không được đưa lại Redis, dù process không chết. Ticket vẫn queued trong DB nhưng vắng mặt trong Redis cho tới khi TicketSweeper expire (MATCHING_QUEUE_MAX_WAIT_SECONDS) hoặc client retry cùng Idempotency-Key để đi nhánh replay re-enqueue. Chưa tìm thấy test hay doc nói về cửa sổ này.
- SafetyService.canPair được gọi BÊN TRONG transaction tryPair đang giữ FOR UPDATE hai ticket (matcher-worker.service.ts:277) và cả trong precheckAccept đang khoá hàng invite (invite.service.ts:475). canPair không nhận EntityManager nên chạy trên kết nối pool khác của Safety: mỗi lần ghép một cặp giữ 2 kết nối DB cùng lúc, đáng lưu ý khi tính pool size theo capacity-roadmap.
