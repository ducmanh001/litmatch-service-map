# Service map: Soul Match

id: service-map:soul-match
module: module:soul-match
deployable: core-api
confidence: high

Phòng chat ẩn danh, rating, mutual-like

Sở hữu phòng chat ẩn danh ngắn sau khi Matching ghép cặp: tin nhắn (append-only), rating 2 chiều rude|boring|like (bất biến) và quyết định 'cả hai like → mở Friendship + unlock profile'. KHÔNG sở hữu MatchSession (Matching ghi), Friendship/Conversation (Friend ghi), hồ sơ người dùng (User) và không có entity phòng riêng: phase chatting/rating/closed được derive từ giờ server.

Owns: bảng soul_chat_messages (append-only, seq bigint identity cho cursor, idempotency_key unique, FK tới match_sessions và users); bảng soul_match_ratings (bất biến, unique (session_id, rater_user_id)); không giữ key Redis nào: chỉ PUBLISH lên channel realtime:user:{userId}

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/soul-match/sessions/:id | jwt | Poll trạng thái phòng chat ẩn danh (phase chatting\|rating\|closed, chatEndsAt/ratingEndsAt, verdict CỦA MÌNH, cờ matched); id lạ hoặc không phải thành viên đều trả 404 gộp để không dò được sessionId. |
| e2 | rest | GET | /api/v1/soul-match/sessions/:id/messages | jwt | Liệt kê message ẩn danh theo cursor seq, chỉ trả senderRole me\|partner (không lộ userId đối phương); phòng đã closed trả 409 SOUL_MATCH_CHAT_NOT_OPEN vì lịch sử ẩn danh bị khoá. |
| e3 | rest | POST | /api/v1/soul-match/sessions/:id/messages | jwt | Gửi message ẩn danh: bắt buộc header Idempotency-Key (unique DB, retry trả lại message cũ), throttle 60 req/phút, chỉ khi phase chatting, re-check UserStatus.Banned, khoá hàng MatchSession rồi lưu và publish soul.message sau commit. |
| e4 | rest | POST | /api/v1/soul-match/sessions/:id/rating | jwt | Chốt verdict rude\|boring\|like một lần (cùng verdict là replay, khác verdict là 409); khi cả hai like thì tạo Friendship + Conversation + import lịch sử chat trong CÙNG transaction rồi publish soul.matched. |
| e5 | rest | POST | /api/v1/soul-match/sessions/:id/end | jwt | Rời phòng: Matching đặt MatchSession sang ended cho cả hai (idempotent, trả 204) rồi publish soul.ended cho cả hai; Friendship đã tạo được giữ nguyên. |
| e6 | rest | GET | /api/v1/soul-match/sessions/:id/partner | jwt | Trả PublicProfileDto của đối phương chỉ khi FriendService.areFriends=true; chưa match trả 403 SOUL_MATCH_PARTNER_LOCKED (Friendship là nguồn sự thật của unlock). |
| e7 | socket-emit | EMIT | soul.message | internal | Báo tin nhắn mới cho cả người gửi (senderRole='me') và đối phương ('partner') qua Redis realtime:user:{userId}; chỉ bắn cho message MỚI (replay idempotency không bắn lại), best-effort, socket đã xác thực JWT ở gateway. |
| e8 | socket-emit | EMIT | soul.matched | internal | Báo cả hai đã cùng like và Friendship vừa được tạo MỚI để client mở khoá profile; không bắn lại khi replay hoặc khi cặp đã là bạn từ session trước. |
| e9 | socket-emit | EMIT | soul.ended | internal | Báo một thành viên đã rời phòng (reason=member_left, không kèm userId) để client đối phương đóng UI ngay; bắn lại ở mỗi lần gọi POST /end, kể cả khi session đã ended. |

## Coordination edges (why this service talks to others)

### đọc/đóng MatchSession (c1) with matching [service, out, di-call]

- endpoints: e1, e2, e3, e4, e5, e6
- what: Đọc MatchSession qua MatchingService.findSessionById để dựng phòng (matchType=soul, status confirmed|ended, confirmedAAt/BAt) và gọi MatchingService.endSoulSession khi một bên rời; Soul Match không bao giờ tự ghi match_sessions.
- why: Phòng Soul không có entity riêng mà là view dẫn xuất từ MatchSession, chỉ Matching được ghi (boundary AGENTS.md/docs/03). Matching cũng là nơi duy nhất ghép cặp (canPair, gender filter) và chốt trạng thái confirmed, nên phòng chỉ mở được khi session đã đủ 2 bên xác nhận.
- if missing: Soul Match phải tự giữ bản sao cặp/trạng thái hoặc ghi thẳng match_sessions: trạng thái lệch với Matching, phá ownership, và không xác minh được ai là thành viên hay phòng đã ended.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:434, apps/core-api/src/modules/soul-match/soul-match.service.ts:105, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:281
- review: sound

### khoá hàng match_sessions (c2) with matching [service, out, shared-db]

- endpoints: e3, e4
- what: Trong sendMessage và rate, mở transaction rồi manager.findOne(MatchSession, lock pessimistic_write) để khoá hàng match_sessions (chỉ khoá, không ghi); các bảng soul_* cũng có FK tới match_sessions(id).
- why: READ COMMITTED: hai rating 'like' song song không thấy insert chưa commit của nhau nên cả hai kết luận 'chưa đủ 2 like' và không ai tạo Friendship; khoá hàng session serialize hai rater. sendMessage dùng cùng khoá để tin nhắn không lọt giữa bước snapshot lịch sử và bước import sang Friend Chat.
- if missing: Mutual-like có thể bị mất (cả hai thích mà không có Friendship) hoặc tin nhắn gửi đúng lúc import bị rơi khỏi lịch sử Friend Chat.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:311, apps/core-api/src/modules/soul-match/soul-match.service.ts:163, apps/core-api/src/database/migrations/1752400000000-soul-match.ts:48
- review: sound

### ensureFriendship (cùng tx) (c3) with friend [service, out, di-call]

- endpoints: e1, e3, e4, e6, e8
- what: rate() gọi FriendService.ensureFriendship(manager, ..., FriendshipSource.SoulMatch, messageSeeds) khi cả hai like, kèm snapshot tin nhắn ẩn danh; FriendService.areFriends quyết định cờ matched, phase chatting vô hạn và quyền xem profile đối phương.
- why: Friendship + Conversation thuộc Friend (bất biến có Friendship ⇒ có Conversation, unique cặp canonical), nên Soul Match phải truyền EntityManager để rating, Friendship, Conversation và lịch sử chat commit/rollback cùng nhau. Friendship là nguồn sự thật duy nhất của 'unlock'.
- if missing: Mutual-like có thể để lại Friendship thiếu Conversation hoặc mất lịch sử chat; cờ matched/unlock lệch giữa hai module; cặp match lại hoặc retry tạo bản ghi trùng.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:362, apps/core-api/src/modules/soul-match/soul-match.service.ts:412, apps/core-api/src/modules/friend/friend.service.ts:124
- review: sound

### re-check ban, public DTO (c4) with user [service, out, di-call]

- endpoints: e3, e6
- what: sendMessage đọc UserService.getByIdOrThrow để chặn UserStatus.Banned ngay lúc gửi; getPartnerProfile trả PublicProfileDto.from(user) (DTO công khai của User) cho đối phương sau khi match.
- why: Access token là JWT stateless TTL ngắn, trạng thái banned chỉ được kiểm tra lại ở refresh/login nên user bị ban giữa phòng vẫn gửi tiếp được tới khi token hết hạn; Soul Match re-check đúng lúc hành động (docs/10 § 10.0.C). Reveal dùng đúng PublicProfileDto để không tự chế DTO lộ field.
- if missing: Tài khoản bị ban vẫn nhắn được trong phòng ẩn danh; profile reveal dễ lệch hoặc lộ field so với DTO công khai chuẩn của User.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:137, apps/core-api/src/common/guards/jwt-auth.guard.ts:18, apps/core-api/src/modules/soul-match/soul-match.service.ts:419
- review: sound

### phát soul.* sau commit (c5) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e3, e4, e5, e7, e8, e9
- what: Sau khi DB commit, publish envelope soul.message / soul.matched / soul.ended lên Redis channel realtime:user:{userId} của từng người nhận; signaling-gateway chỉ relay nguyên văn xuống socket Socket.IO.
- why: Gateway không có DB hay business logic nên mọi quyết định ẩn danh phải xong ở core-api: payload tính sẵn theo từng người nhận (senderRole me|partner, không có userId). Publish sau commit để không bắn event của giao dịch bị rollback; best-effort vì REST poll vẫn là nguồn sự thật.
- if missing: Client chỉ biết có tin nhắn mới, mutual-like hoặc đối phương rời phòng khi poll: chat ngắn bị trễ và UI của đối phương không đóng ngay khi một bên rời.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:215, apps/core-api/src/modules/soul-match/soul-match.service.ts:112, apps/signaling-gateway/src/app/signaling.gateway.ts:346
- review: sound

### PUBLISH realtime:user (c6) with redis [infra, out, redis-pubsub]

- endpoints: e7, e8, e9
- what: Module giữ một client ioredis riêng (SOUL_MATCH_REDIS, tạo từ REDIS_URL, đóng ở onApplicationShutdown) và chỉ dùng để PUBLISH lên realtime:user:{userId}; không lưu key hay queue nào.
- why: Gateway là process khác nên đường duy nhất từ core-api sang gateway là Redis pub/sub; pub/sub không lưu và không bảo đảm giao, vì vậy event chỉ là gợi ý refetch còn REST + DB mới là sự thật. Client riêng để publish không dùng chung connection.
- if missing: Không có kênh đẩy sang gateway; mọi cập nhật chỉ còn REST polling.
- evidence: apps/core-api/src/modules/soul-match/redis/soul-match-redis.provider.ts:7, apps/core-api/src/modules/soul-match/redis/soul-match-redis.provider.ts:14, apps/core-api/src/modules/soul-match/soul-match.module.ts:34
- review: sound

### tin nhắn + rating bất biến (c7) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6
- what: Ghi soul_chat_messages (append-only, seq bigint identity) và soul_match_ratings trong transaction của core-api; unique DB uq_soul_chat_messages_idempotency_key và uq_soul_match_ratings_session_rater là chốt chặn cuối.
- why: Idempotency và race của chat/rating không thể dựa vào check-rồi-insert ở code: retry mạng và hai request song song chỉ an toàn nhờ unique constraint DB. Message giữ append-only (khoá qua API, không xoá) để còn bằng chứng cho Report/T&S.
- if missing: Retry nhân đôi tin nhắn, một người có hai verdict, và report 'thô lỗ' không còn evidence để xử lý.
- evidence: apps/core-api/src/modules/soul-match/entities/soul-chat-message.entity.ts:14, apps/core-api/src/modules/soul-match/entities/soul-match-rating.entity.ts:23, apps/core-api/src/database/migrations/1752400000000-soul-match.ts:53
- review: sound

### poll, Idempotency-Key, /end (c8) with web [client, in, http]

- endpoints: e1, e2, e3, e4, e5, e6
- what: Web poll GET session/messages làm fallback, tự sinh Idempotency-Key cho mỗi tin nhắn (chỉ reset sau khi gửi thành công) và gọi POST /end khi user rời màn (pagehide, popstate, click link).
- why: Server không có presence hay job nào tự kết thúc phòng: MatchSession chỉ sang ended qua POST /end nên client phải gọi khi rời. Idempotency-Key do client sinh vì chỉ client biết lần gửi lại là retry; realtime là best-effort nên client bắt buộc có poll fallback.
- if missing: Không gọi /end thì phòng của đối phương không đóng ngay (chỉ tự hết theo đồng hồ, hoặc mở vô thời hạn nếu đã match); key đổi mỗi lần retry thì tin nhắn bị nhân đôi.
- evidence: apps/web/src/features/soul-match/api.ts:70, apps/web/src/features/soul-match/components/soul-message-composer.tsx:28, apps/web/src/features/soul-match/components/soul-chat-phase-view.tsx:60
- review: sound

### soul.* → refetch (c9) with web [client, out, socket]

- endpoints: e7, e8, e9
- what: Event soul.message / soul.matched / soul.ended đến web qua socket của signaling-gateway; web chỉ dùng chúng để invalidate và refetch query tương ứng (messages hoặc session), soul.ended còn đặt lý do đóng phòng ngay.
- why: Event là gợi ý best-effort, không mang đủ state để thay REST, nên client phải hiểu 'nhận event ⇒ refetch REST'. senderRole đã tính sẵn theo từng người nhận nên web render được mà không cần biết userId đối phương.
- if missing: Client bỏ qua event thì chat chỉ cập nhật theo chu kỳ poll; client coi event là nguồn sự thật thì một event bị mất (best-effort) làm lệch trạng thái phòng.
- evidence: apps/web/src/features/soul-match/components/soul-chat-phase-view.tsx:70, apps/web/src/features/soul-match/components/soul-chat-phase-view.tsx:71, apps/web/src/features/soul-match/components/soul-chat-phase-view.tsx:85
- review: sound

## Notes

- Đã đối chiếu docs/services/soul-match-service.md với code: 6 route, 3 event (soul.message/soul.matched/soul.ended), phase derive-khi-đọc, rating bất biến, import lịch sử vào Friend Chat đều khớp; không thấy lệch.
- Soul Match KHÔNG import Safety: block/report chỉ được chặn ở Matching lúc ghép cặp (canPair qua MATCH_INTERACTION_POLICY). Trong phòng chỉ re-check UserStatus.Banned; không thấy kiểm block/report khi gửi message, kể cả sau khi đã matched (phòng giữ phase chatting tới khi POST /end). Người kiểm chứng nên xác nhận đây là chủ đích.
- Không có job/cron: confirmedAt = max(confirmedAAt, confirmedBAt); chatEndsAt = confirmedAt + SOUL_CHAT_DURATION_SECONDS (mặc định 150); ratingEndsAt = chatEndsAt + SOUL_RATING_WINDOW_SECONDS (mặc định 120). Trạng thái ended chỉ được đặt bởi MatchingService.endSoulSession, và chỉ SoulMatchService.endSession gọi hàm này.
- POST /end publish lại soul.ended cho cả hai ở MỖI lần gọi (kể cả khi session đã ended), khác soul.message/soul.matched chỉ bắn ở lần tạo đầu; client phải xử lý event này idempotent.
- SoulMatchModule exports rỗng và không module nào import nó; cũng không module nào (Safety/Admin) đọc soul_chat_messages làm bằng chứng T&S: 'giữ cho T&S' hiện chỉ là ý định lưu trữ, chưa có call site đọc.
- Bất biến ẩn danh dựa vào kỷ luật phía gọi: GET /users/:id (module user) trả PublicProfileDto cho bất kỳ userId nào, nên ẩn danh chỉ giữ được vì Matching/Soul Match không bao giờ trả userId đối phương trước khi cả hai like (payload match_confirmed chỉ có sessionId + matchType).
- Import lịch sử chat dùng key friend:import:soul:{sessionId}:{messageId} (soul-match.constants.ts) trùng định dạng với migration backfill 1756600000000; unique messages.idempotency_key của Friend làm retry/replay là no-op.
