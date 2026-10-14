# Service map: Signaling Gateway (Socket.IO)

id: service-map:signaling-gateway
module: module:signaling-gateway
deployable: signaling-gateway
confidence: high

Fanout Socket.IO, quota kết nối, presence

Gateway Socket.IO namespace /signaling: verify JWT của core-api lúc handshake, join socket vào room user:{userId}, relay nguyên văn envelope từ Redis pub/sub realtime:user:* xuống socket, và giữ quota kết nối + presence bằng lease trong Redis. KHÔNG chứa business logic, KHÔNG có DB, KHÔNG ký token và KHÔNG nhận room/event từ client (chỉ có 'ping').

Owns: Redis ZSET signaling:connection-quota:{userId} (lease mỗi socket, TTL WS_CONNECTION_LEASE_MS); Redis ZSET realtime:presence:{userId} (cùng lease, core-api đọc để biết online); Room Socket.IO user:{userId} (in-memory, đồng bộ giữa pod bằng Redis adapter)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /health | public | Liveness tương thích ngược, trả status ok + uptimeSeconds; không kiểm tra Redis. |
| e2 | rest | GET | /health/live | public | Liveness probe của K8s: chỉ cho biết process còn sống, không phụ thuộc Redis. |
| e3 | rest | GET | /health/ready | public | Readiness probe: 503 nếu PSUBSCRIBE realtime:user:* chưa sẵn sàng, quota Redis chưa ready hoặc Redis cluster adapter chưa ready (checks.redisSubscription / redisClusterAdapter). |
| e50 | socket-listen | CONNECT | /signaling | jwt | Handshake Socket.IO: middleware verify handshake.auth.token (JWT của core-api), gán userId từ payload.sub, acquire lease quota (tối đa WS_MAX_CONNECTIONS_PER_USER) rồi mới cho thành socket; sai token -> connect_error UNAUTHORIZED, hết quota -> CONNECTION_LIMI… |
| e51 | socket-listen | EVENT | ping | jwt | Client gửi 'ping' nhận lại 'pong': kiểm tra kết nối ở tầng ứng dụng; đây là message duy nhất gateway nhận từ client (không có join/leave room theo yêu cầu). |
| e52 | socket-emit | EMIT | {event} -> room user:{userId} | jwt | Relay: nhận envelope từ Redis pmessage trên realtime:user:{userId}, parse userId từ tên channel và emit(envelope.event, envelope.data) vào room của đúng user, không đọc/sửa payload; channel lạ hoặc JSON hỏng bị bỏ qua. |

## Coordination edges (why this service talks to others)

### gift.sent cho phòng tiệc (c1) with gift [service, in, redis-pubsub]

- endpoints: e52
- what: GiftService sau khi giao dịch kim cương commit gọi publishRealtimeEvent với envelope gift.sent (roomId, giftCode, priceDiamond, pointsAwarded...) cho từng member active của party room.
- why: Mọi member phòng phải thấy hiệu ứng quà ngay, nhưng GiftService không giữ socket nào; nó chỉ publish lên realtime:user:{memberId} (sau transaction tiền) để gateway giữ kết nối đẩy xuống.
- if missing: Quà vẫn trừ/cộng tiền đúng nhưng người trong phòng không thấy hiệu ứng tức thì, phải chờ REST refetch.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:498
- review: sound — publishGiftSent chỉ chạy khi !result.replayed, sau commit, lỗi chỉ warn; publish per-user qua publishRealtimeEvent.

### match.confirmed khi ghép cặp (c2) with matching [service, in, redis-pubsub]

- endpoints: e52
- what: Matcher worker (tryPair sau commit), MatchingService.confirm và InviteService publish match.confirmed {ticketId, sessionId} cho từng user của cặp.
- why: Hai người chờ ghép đang mở UI chờ, cần biết phiên đã chốt ngay; core-api chỉ publish sau khi DB commit (best-effort) và để gateway fanout, vì worker nền không có kết nối tới trình duyệt.
- if missing: UI chờ ghép chỉ biết kết quả qua polling GET /matching/tickets/:id, trễ hơn nhiều giây.
- evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:377, apps/core-api/src/modules/matching/matching.module.ts:86
- review: sound — matcher-worker, MatchingService.confirm và InviteService đều publish match.confirmed sau khi transaction commit.

### call.ended cho 2 bên (c3) with calling [service, in, redis-pubsub]

- endpoints: e52
- what: CallingService ở mọi nhánh kết thúc cuộc gọi, sau commit và deleteRoom LiveKit, publish call.ended {callId, matchSessionId, reason, durationSeconds} cho userAId và userBId.
- why: Khi cuộc gọi kết thúc (ở bất kỳ nhánh nào), bên còn lại phải đóng UI cuộc gọi ngay; chỉ gateway biết socket của họ nên core-api phải đi qua kênh realtime:user:{uid}.
- if missing: Bên kia vẫn nhìn màn hình gọi cho tới khi LiveKit ngắt hoặc client poll, gây trải nghiệm 'gọi ma'.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:710
- review: sound — cleanupEndedCall chạy sau endById commit, sau endVoiceSessionForCall và deleteRoom, rồi publish call.ended cho userAId/userBId.

### soul.message/matched/ended (c4) with soul-match [service, in, redis-pubsub]

- endpoints: e52
- what: SoulMatchService publish soul.message (payload per-recipient senderRole me|partner), soul.matched khi mutual like tạo friendship, soul.ended khi một bên rời phòng, cho cả hai user của session.
- why: Chat ẩn danh phải đẩy tin tức thì nhưng payload tính sẵn theo người nhận (không bao giờ chứa userId đối phương); gateway chỉ relay nguyên văn nên việc ẩn danh phải được quyết ở core-api lúc publish.
- if missing: Chat ẩn danh phải polling /soul-match/sessions/:id/messages; mất tín hiệu match/rời phòng tức thì.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:112, apps/core-api/src/modules/soul-match/soul-match.service.ts:215
- review: sound — soul.message tính senderRole me/partner per-recipient, payload không có userId đối phương; publish sau persist, replay không bắn lại.

### movie.* trạng thái xem phim (c5) with movie-match [service, in, redis-pubsub]

- endpoints: e52
- what: MovieMatchService publish movie.session.started, movie.state.changed, movie.session.ended, movie.reaction.sent cho hai user (userLowId/userHighId) của phiên.
- why: Xem phim đồng bộ cần mỗi thao tác play/pause/reaction tới bạn xem gần như tức thì; trạng thái bền nằm DB ở core-api còn gateway chỉ là đường đẩy một chiều.
- if missing: Hai người xem lệch trạng thái play/pause và không thấy reaction cho đến khi refetch.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:182, apps/core-api/src/modules/movie-match/movie-match.service.ts:232
- review: sound — movie.session.started/state.changed/ended/reaction.sent đều publish tới cặp user; state bền ở DB.

### minigame.session.* (c6) with mini-game [service, in, redis-pubsub]

- endpoints: e52
- what: MiniGameService publish minigame.session.started {sessionId, gameType, initiatorUserId} và minigame.session.resolved cho hai user.
- why: Người được mời chơi phải thấy lời mời/kết quả ván ngay; core-api chốt kết quả trong DB rồi mới publish nên gateway không cần biết luật game.
- if missing: Đối thủ không biết ván bắt đầu hoặc đã có kết quả cho tới khi poll.
- evidence: apps/core-api/src/modules/mini-game/mini-game.service.ts:130, apps/core-api/src/modules/mini-game/mini-game.service.ts:232
- review: sound — minigame.session.resolved publish sau khi UPDATE có điều kiện chốt kết quả trong DB.

### friend.message / streak (c7) with friend [service, in, redis-pubsub]

- endpoints: e52
- what: FriendService publish friend.message sau khi lưu tin chat 1-1 và friend.streak.increased khi chạm mốc; job streak-warning publish friend.streak.at_risk cho cả hai bạn.
- why: Tin chat và cảnh báo streak sắp đứt cần tới cả hai bạn ngay, kể cả job nền không có socket; do đó đi qua kênh theo người nhận và gateway fanout.
- if missing: Tin nhắn bạn bè không hiện tức thì (phải mở lại/refetch) và cảnh báo streak sắp đứt không tới kịp.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:399, apps/core-api/src/modules/friend/friend.service.ts:425
- review: sound — friend.message/streak.increased publish sau khi lưu; streak-warning.job publish friend.streak.at_risk.

### party.* sự kiện phòng (c8) with party-room [service, in, redis-pubsub]

- endpoints: e52
- what: PartyRoomService publish party.member.joined/left/disconnected/reconnected, party.role.changed, party.speaker.invite.received, party.comment.created, party.host.disconnected/reconnected, party.room.closed cho member active (hoặc đúng người được mời).
- why: Danh sách member, quyền speaker và việc đóng phòng thay đổi liên tục; core-api quyết ai là member lúc publish (authz theo recipient) rồi gateway fanout, nên client không được tự join room.
- if missing: UI phòng tiệc không cập nhật member/role/bình luận tức thì và không biết phòng đã đóng cho tới khi poll REST.
- evidence: apps/core-api/src/modules/party-room/party-room.service.ts:559, apps/core-api/src/modules/party-room/party-room.service.ts:1058
- review: sound — publishToRoomMembers liệt kê member active tại thời điểm publish; invite chỉ gửi tới target.

### PSUBSCRIBE realtime:user:* (c50) with redis [infra, in, redis-pubsub]

- endpoints: e52
- what: Một connection ioredis riêng ở subscriber mode gọi psubscribe(REALTIME_USER_CHANNEL_PATTERN); mỗi pmessage được relay() đổi thành emit vào room user:{userId}.
- why: Gateway không có DB và không có API nội bộ với core-api, nên Redis pub/sub là kênh duy nhất nhận event; connection phải riêng vì ioredis ở subscriber mode không chạy được lệnh khác.
- if missing: Không có đường nào đưa event của core-api xuống socket; /health/ready trả 503 khi pattern chưa subscribe nên pod không nhận traffic.
- evidence: apps/signaling-gateway/src/app/signaling.gateway.ts:346
- review: sound — Subscriber ioredis riêng psubscribe(REALTIME_USER_CHANNEL_PATTERN); isReady() đòi subscriptionReady nên /health/ready trả 503 khi chưa subscribe.

### lease quota + presence (c51) with redis [infra, out, redis-state]

- endpoints: e50
- what: Lua script ACQUIRE/REFRESH/RELEASE ghi ZSET signaling:connection-quota:{userId} và realtime:presence:{userId}: mỗi socket một lease có TTL WS_CONNECTION_LEASE_MS, refresh mỗi leaseMs/3, release khi disconnect.
- why: Giới hạn tối đa N socket/user phải đúng trên toàn cụm replica nên cần đếm atomic ở Redis (Lua), không dùng Map cục bộ; TTL để pod chết không giữ slot vĩnh viễn, và cùng lease đó làm tín hiệu presence.
- if missing: Quota chỉ đúng trong từng pod nên user mở nhiều tab vượt giới hạn; Redis lỗi thì handshake fail closed (CONNECTION_QUOTA_UNAVAILABLE) và socket đang sống bị ngắt.
- evidence: apps/signaling-gateway/src/app/connection-quota.service.ts:15, apps/signaling-gateway/src/app/connection-quota.service.ts:126
- review: sound — Lua ACQUIRE/REFRESH/RELEASE ghi cả quota ZSET và presence ZSET, refresh mỗi leaseMs/3, Redis lỗi thì fail closed và ngắt socket đang sống.

### Socket.IO Redis adapter (c52) with redis [infra, both, sdk]

- endpoints: e50, e52
- what: SignalingRedisAdapterService mở 2 client Redis (pub + sub) và createAdapter(pubClient, subClient) từ @socket.io/redis-adapter, gắn vào CorsIoAdapter trước app.listen() (main.ts).
- why: Để server.to(room).emit() ở pod này tới được socket ở pod khác khi scale ngang; phải connect xong trước listen vì afterInit() cần Server đã có adapter. Lưu ý: relay() cũng dùng server.to(...).emit() và mỗi pod đều PSUBSCRIBE, nên với adapter có thể một event tới client nhiều lần (xem finding V1).
- if missing: Mỗi pod chỉ thấy room của chính nó: broadcast/fetchSockets xuyên pod không dùng được và /health/ready trả 503 (redisClusterAdapter down). Tách biệt: relay PSUBSCRIBE tự nó đã đủ phát cho room cục bộ; adapter không cần cho relay mà còn có nguy cơ nhân đôi event.
- evidence: apps/signaling-gateway/src/app/redis-adapter.service.ts:46
- review: corrected — Phần what/why đúng; ifMissing cần bỏ ý 'relay vẫn chạy' và nêu rủi ro trùng event khi có adapter (xem mục "Điểm đáng chú ý").

### đọc presence online (c53) with user [service, in, redis-state]

- endpoints: e50
- what: RedisPresenceReader của user module đọc ZSET realtime:presence:{userId} (xoá member hết hạn rồi zcard > 0) để trả isOnline cho GET /users/:id/presence, chỉ khi privacy showOnlineStatus bật.
- why: Core-api không giữ socket nên không thể tự biết user có online; gateway là nơi duy nhất thấy kết nối thật và công bố qua key Redis dùng chung (cùng helper realtimePresenceKey trong common-dtos).
- if missing: Trạng thái online luôn false hoặc phải gateway có API nội bộ riêng; hiện đọc lỗi thì fail closed trả offline thay vì online giả.
- evidence: apps/core-api/src/modules/user/redis/redis-presence-reader.adapter.ts:15, apps/core-api/src/modules/user/user.controller.ts:81
- review: sound — RedisPresenceReader fail closed; user.controller chỉ gọi presence.isOnline khi settings.showOnlineStatus bật.

### chung JWT_SECRET (c54) with auth [service, in, config]

- endpoints: e50
- what: Gateway đăng ký JwtModule với JWT_SECRET (Joi min 32, bắt buộc) chỉ để verifyAsync access token do core-api auth module ký bằng cùng secret; lấy userId từ payload.sub. [10/10] Từ commit 80106a1/54c66ab có xoay khoá không đăng xuất mọi người: core-api và gateway ký bằng JWT_SECRET, chỉ khi CHỮ KÝ không khớp mới thử JWT_SECRET_PREVIOUS (token hết hạn hay sai claim không được thử lại); quy trình 3 bước vì rollout cuốn chiếu làm pod cũ từ chối token do pod mới ký; env validation từ chối JWT_SECRET_PREVIOUS trùng AUTH_GUEST_DEVICE_TOKEN_SECRET.
- why: Gateway không có DB hay API nội bộ nên cách duy nhất để biết socket thuộc user nào là tự verify chữ ký token; vì vậy hai deployable bắt buộc cùng secret, và gateway không bao giờ được ký token.
- if missing: Lệch secret làm mọi handshake bị connect_error UNAUTHORIZED (realtime chết, client chỉ còn polling); ngược lại nếu gateway tin token không verify thì ai cũng nghe được event của user khác.
- evidence: apps/signaling-gateway/src/app/app.module.ts:35, apps/core-api/src/modules/auth/auth.module.ts:32
- review: sound — env.validation: JWT_SECRET Joi.string().min(32).required(); gateway chỉ verifyAsync, lấy payload.sub.

### kết nối và nhận event (c55) with web [client, both, socket]

- endpoints: e50, e51, e52
- what: Web mở một socket io(NEXT_PUBLIC_SOCKET_URL + '/signaling') với auth callback lấy access token tươi mỗi lần (re)connect, transports websocket; khi connect_error UNAUTHORIZED thì refreshSession rồi connect lại; sau reconnect refetch state qua REST.
- why: Socket chỉ là kênh delta best-effort nên client phải tự phục hồi: gửi đúng namespace /signaling và token còn hạn, refresh khi hết hạn, và resync REST sau reconnect vì event bị miss không được phát lại.
- if missing: Sai namespace thì client rơi vào namespace / không ai lắng nghe mà không báo lỗi; token hết hạn không refresh thì realtime ngắt vĩnh viễn; không resync thì UI lệch state sau mất mạng.
- evidence: apps/web/src/shared/realtime/socket.ts:44, apps/web/src/shared/realtime/socket.ts:46
- review: sound — socket.ts: namespace /signaling, auth callback token tươi, connect_error Unauthorized gọi refreshSession, onReconnected invalidate REST.

## Notes

- specs/realtime.asyncapi.yaml chỉ liệt kê 8 event (soul.message, match.confirmed, call.ended, friend.message, party.room.closed, gift.sent, movie.state.changed, minigame.session.resolved) trong khi RealtimeEvents ở libs/common-dtos/src/lib/realtime-events.ts có 26 tên; file nguồn TS mới là hợp đồng đầy đủ (docs cũng nói vậy).
- docs/services/realtime-gateway.md § 2 liệt kê match.matched nhưng KHÔNG tìm thấy RealtimeEvents.MatchMatched được publish ở bất kỳ nơi nào trong apps/core-api/src (chỉ matching.metrics.ts nhắc trong comment): tên event có trong hợp đồng nhưng hiện chưa có producer. Ngược lại party.member.disconnected/reconnected, party.speaker.invite.received, party.comment.created có producer trong code nhưng chưa nằm trong danh sách event của doc.
- Bảng producer ở docs § 4 không có movie-match và mini-game; code thực tế có cả hai (movie.* và minigame.*), cùng friend.streak.* (kể cả job streak-warning.job.ts) và party.* nhiều hơn doc.
- Doc § 5 nói 'Socket chết được dọn bởi ping/pong mặc định của Socket.IO'; code còn có thêm lease refresh mỗi leaseMs/3 và tự disconnect khi refresh thất bại hoặc Redis quota mất kết nối (fail closed).
- Redis adapter (@socket.io/redis-adapter) hiện KHÔNG cần cho relay: mỗi pod tự PSUBSCRIBE toàn bộ realtime:user:* và emit vào room cục bộ; adapter chỉ mở đường cho broadcast xuyên pod và vẫn nằm trong readiness /health/ready.
- Không có dossier AST cho service này (không phải module core-api): endpoint chính (handshake, ping, relay) được thêm tay từ signaling.gateway.ts.
- Quota tối đa schema ràng buộc 1..3 (WS_MAX_CONNECTIONS_PER_USER, mặc định 3).
