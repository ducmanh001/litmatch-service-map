# Service map: Mini Game

id: service-map:mini-game
module: module:mini-game
deployable: core-api
confidence: high

Oẳn tù tì 2 người giữa bạn bè, move ẩn

Sở hữu phiên oẳn tù tì giữa 2 người đã là bạn (mini_game_sessions + bảng participant giữ bất biến 1 ván chờ move/user) và logic thắng thua. Không sở hữu quan hệ bạn bè (Friend), không chạm Economy, không tích hợp tin nhắn chat; chỉ publish 2 event realtime sau khi commit.

Owns: mini_game_sessions (userLowId/userHighId canonical, lowMove/highMove, status, winnerUserId); mini_game_active_participants (PK userId: 1 user chỉ có 1 ván waiting_moves); ioredis client riêng MINI_GAME_REDIS (chỉ để publish realtime)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/mini-game/sessions | jwt | Tạo ván oẳn tù tì với một người bạn (404 nếu không phải bạn), idempotent theo cặp, 409 ALREADY_WAITING nếu user đang chờ move ở cặp khác; publish minigame.session.started cho cả hai. |
| e2 | rest | POST | /api/v1/mini-game/sessions/:id/moves | jwt | Nộp move rock/paper/scissors bằng UPDATE có điều kiện (move IS NULL AND status=waiting_moves); đủ 2 move thì resolve trong cùng transaction và publish minigame.session.resolved kèm cả 2 move. |
| e3 | rest | POST | /api/v1/mini-game/sessions/:id/cancel | jwt | Một trong hai participant huỷ ván đang chờ move (idempotent nếu đã cancelled, 409 nếu đã resolved), giải phóng slot participant. |
| e4 | rest | GET | /api/v1/mini-game/sessions/:id | jwt | Đọc ván (chỉ participant, 404 gộp nếu không tồn tại/không phải thành viên); là fallback polling khi realtime không tới. |
| e5 | socket-emit | EMIT | minigame.session.resolved | internal | Event minigame.session.resolved gửi cho CẢ HAI user sau khi cả hai nộp move: sessionId, lowMove, highMove, winnerUserId (null nếu hoà). |
| e6 | socket-emit | EMIT | minigame.session.started | internal | Event minigame.session.started gửi cho CẢ HAI user khi ván được tạo: sessionId, gameType, initiatorUserId. |

## Coordination edges (why this service talks to others)

### xác minh là bạn bè (c1) with friend [service, out, di-call]

- endpoints: e1
- what: createSession gọi FriendService.areFriends(userId, friendUserId) trước khi tạo MiniGameSession; không phải bạn thì trả 404 MINI_GAME_NOT_FRIEND.
- why: Mini game tái dùng quan hệ Friendship thay vì dựng khái niệm 'mời chơi' riêng (docs mini-game § 1); Friend là chủ sở hữu bảng friendships nên chỉ nó trả lời được 2 người có là bạn không. Dùng chung 404 cho 'không tồn tại' và 'không phải bạn' để không lộ quan hệ.
- if missing: Bất kỳ user nào cũng tạo được ván và đẩy event minigame.session.started tới người lạ biết userId của họ.
- evidence: apps/core-api/src/modules/mini-game/mini-game.service.ts:71
- review: sound

### đẩy event minigame.* (c2) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e5, e6
- what: Sau transaction commit, publishRealtimeEvent đẩy envelope minigame.session.started (khi tạo) và minigame.session.resolved (kèm lowMove/highMove/winnerUserId) lên kênh realtime:user:{userId} của từng participant; signaling-gateway PSUBSCRIBE rồi relay xuống socket.
- why: Hai người chơi đồng thời nên người nộp move trước cần biết khi nào đối phương nộp xong và kết quả; core-api không giữ socket, chỉ gateway giữ nên phải qua Redis. Không có event trung gian 'một bên đã nộp' để channel khác không suy ra ai nộp trước; payload gửi cùng lúc cho cả hai.
- if missing: Client phải poll GET /mini-game/sessions/:id liên tục để thấy ván bắt đầu và kết quả; trải nghiệm chơi đồng thời bị trễ.
- evidence: apps/core-api/src/modules/mini-game/mini-game.service.ts:130, apps/core-api/src/modules/mini-game/mini-game.service.ts:232
- review: sound

### client ioredis publish (c3) with redis [infra, out, redis-pubsub]

- endpoints: e1, e2
- what: MiniGameModule tạo ioredis riêng MINI_GAME_REDIS từ REDIS_URL (createCoreRedisClient) để publish; đóng bằng closeCoreRedisClient khi onApplicationShutdown.
- why: Redis pub/sub là đường duy nhất core-api nói chuyện với signaling-gateway (gateway không có DB và không nhận HTTP từ core-api); client riêng được tạo và đóng theo vòng đời module để không rò kết nối khi tắt app.
- if missing: Không có kết nối Redis nên không publish được minigame.session.*; kết nối không được đóng gây treo shutdown/rò socket.
- evidence: apps/core-api/src/modules/mini-game/mini-game.module.ts:35
- review: sound

### ván + slot participant (c4) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4
- what: Ghi mini_game_sessions (lowMove/highMove, status, winnerUserId) và mini_game_active_participants; submitMove dùng UPDATE ... WHERE move IS NULL AND status='waiting_moves' trong transaction rồi đọc lại để resolve.
- why: Hai request nộp move đồng thời và double-submit là tình huống chính của game 2 người; chỉ UPDATE có điều kiện ở DB mới chặn đổi move đã nộp, và PK userId của participant chặn một user có 2 ván chờ move cùng lúc, kể cả khi 2 instance core-api chạy song song.
- if missing: Người chơi sửa được move sau khi thấy đối phương, một user có nhiều ván song song, hoặc resolve 2 lần với kết quả khác nhau.
- evidence: apps/core-api/src/modules/mini-game/entities/mini-game-active-participant.entity.ts:19, apps/core-api/src/modules/mini-game/entities/mini-game-session.entity.ts:25
- review: sound

## Notes

- createSession chỉ kiểm tra FriendService.areFriends, không kiểm tra block: nếu Safety block mà Friendship vẫn còn thì vẫn tạo được ván (suy luận từ code: không có import SafetyModule trong luồng này). Doc cũng chỉ nói verify bạn bè.
- submitMove/cancel/get chỉ kiểm tra caller là participant của session, không kiểm tra lại friendship; hợp lệ vì friendship được kiểm tra lúc tạo ván.
- Không có timer hay job: ván waiting_moves bị bỏ dở giữ slot uq participant của cả 2 user cho đến khi một bên gọi POST /sessions/:id/cancel (đúng như doc § 6 nói 'chưa có hết hạn').
- Không thấy client apps/web nghe minigame.session.* hoặc gọi /mini-game/* trong source (grep không có kết quả), nên không thêm collaborator client; fallback polling được thiết kế qua GET /sessions/:id.
- Doc nói best-effort + fallback polling; code khớp: publishRealtimeEvent nuốt lỗi và log warn, và publish chạy SAU transaction commit.
