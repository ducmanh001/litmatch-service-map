# Service map: Palm Match

id: service-map:palm-match
module: module:palm-match
deployable: core-api
confidence: high

Ghép đôi ẩn danh theo hàng đợi, bói bài

Sở hữu hàng đợi ghép cặp ẩn danh Palm Match, phiên 2 người (lật bài, độ hợp, đánh giá like/skip) và kho nội dung bói palm_reading_templates. Không sở hữu quan hệ bạn bè/Conversation (Friend tạo), không sở hữu block/report (Safety quyết định), không chạm Economy và không phát realtime.

Owns: palm_match_sessions (snapshot dấu hiệu, độ hợp, fortune, rating, outcome, expiresAt); palm_match_queue_entries (hàng đợi, PK userId); palm_match_active_participants (PK userId: 1 user chỉ ở 1 phiên active); palm_reading_templates (catalog nội dung bói); advisory lock Postgres 'litmatch:palm-match:pairing'

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/palm-match/reading | jwt | Trả nội dung bói giải trí {category, content, forDate}, deterministic theo hash(userId, category, ngày UTC); chỉ cần JWT, không lưu gì. |
| e2 | rest | POST | /api/v1/palm-match/queue | jwt | Vào hàng đợi (idempotent) và thử ghép ngay với entry cũ nhất còn hợp lệ; mỗi ứng viên phải qua SafetyService.canPair, serialize bằng advisory lock. |
| e3 | rest | GET | /api/v1/palm-match/current | jwt | State phục hồi/poll của queue hoặc phiên hiện tại, đồng thời lazy-expire hạn queue/phiên; là kênh DUY NHẤT để client biết đã được ghép. |
| e4 | rest | POST | /api/v1/palm-match/sessions/:id/flip | jwt | Lật lá của chính caller (idempotent), quyền theo PalmMatchActiveParticipant(userId, sessionId) lấy từ JWT, khoá phiên pessimistic_write. |
| e5 | rest | POST | /api/v1/palm-match/sessions/:id/rating | jwt | Chốt like/skip một lần sau khi cả hai đã lật; cả hai like thì tạo Friendship + Conversation trong cùng transaction qua FriendService.ensureFriendship. |
| e6 | rest | DELETE | /api/v1/palm-match/current | jwt | Huỷ queue, huỷ phiên đang active cho cả hai hoặc dismiss kết quả terminal để giải phóng slot enqueue lượt mới. |

## Coordination edges (why this service talks to others)

### mutual-like tạo bạn (c1) with friend [service, out, di-call]

- endpoints: e5
- what: Khi lowRating và highRating đều là Like, rate() gọi FriendService.ensureFriendship(manager, userLowId, userHighId, FriendshipSource.PalmMatch) với EntityManager của transaction rating.
- why: Friendship và Conversation 1:1 chỉ do Friend sở hữu và phải được tạo atomically cùng nhau (bất biến Friendship ⇒ Conversation). Truyền manager đảm bảo việc chốt outcome Matched của phiên và việc mở bạn là một DB transaction.
- if missing: Phiên có thể ghi Matched mà không có Friendship (hoặc ngược lại có Friendship nhưng thiếu Conversation), người dùng 'match' xong không chat được hoặc bị lệch trạng thái khi retry.
- evidence: apps/core-api/src/modules/palm-match/palm-match.service.ts:286
- review: sound

### kiểm tra block/report (c2) with safety [service, out, di-call]

- endpoints: e2
- what: Với mỗi ứng viên trong 20 entry cũ nhất, joinQueue gọi SafetyService.canPair(userId, candidate.user_id): từ chối nếu có block active 2 chiều hoặc report/block trong SAFETY_REMATCH_COOLDOWN_DAYS.
- why: Ghép ẩn danh nên hai người không tự lọc được nhau; Safety là chủ sở hữu block/report và theo docs/10 § 10.0.C phải kiểm tra lại đúng lúc ghép chứ không tin trạng thái lúc enqueue (user có thể bị block/report khi đang chờ).
- if missing: Người đã block hoặc từng report nhau trong cooldown vẫn bị ghép lại với nhau, phá cam kết an toàn của ghép ẩn danh.
- evidence: apps/core-api/src/modules/palm-match/palm-match.service.ts:158
- review: sound

### queue/phiên + advisory lock (c3) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6
- what: Ghi palm_match_sessions, palm_match_queue_entries, palm_match_active_participants và đọc palm_reading_templates; joinQueue/dismissCurrent lấy pg_advisory_xact_lock(hashtext('litmatch:palm-match:pairing')) trước khi chọn cặp.
- why: Nhiều request ghép chạy song song trên nhiều instance core-api: nếu không serialize việc chọn cặp thì 2 user có thể cùng bắt 1 ứng viên. Lock Postgres cộng PK userId của queue/active-participant biến trạng thái ghép cặp thành bất biến ở DB, không phụ thuộc bộ nhớ process.
- if missing: Một user có thể bị ghép vào 2 phiên cùng lúc hoặc 2 request tạo 2 phiên cho cùng cặp; trạng thái hàng đợi mất khi restart.
- evidence: apps/core-api/src/modules/palm-match/entities/palm-match-session.entity.ts:26, apps/core-api/src/modules/palm-match/entities/palm-match-session.entity.ts:76, apps/core-api/src/modules/palm-match/entities/palm-match-session.entity.ts:89
- review: sound

### poll /current khi đang chờ (c50) with web [client, in, http]

- endpoints: e2, e3, e6
- what: apps/web gọi POST /palm-match/queue để vào hàng đợi rồi lặp GET /palm-match/current (react-query refetchInterval) trong khi state còn queued hoặc session đang active; DELETE /current để huỷ.
- why: Palm Match không có socket-emit nào nên server không đẩy được thông báo 'đã ghép'; client bắt buộc poll /current để thấy phiên mới, trạng thái lật bài của đối phương và kết quả.
- if missing: Người dùng xếp hàng xong không bao giờ biết mình đã được ghép hay đối phương đã lật/rating, màn hình đứng ở trạng thái chờ.
- evidence: apps/web/src/features/palm-match/api.ts:30
- review: sound

## Notes

- Doc docs/services/palm-match-service.md lỗi thời: chỉ mô tả GET /palm-match/reading, trong khi code còn có hàng đợi ẩn danh (POST /queue, GET/DELETE /current), POST /sessions/:id/flip và /rating, 3 bảng session/queue/active-participant và mutual-like mở Friendship (FriendshipSource.PalmMatch). Theo code.
- Doc nói 'không có bảng lưu lịch sử' và 'guest dùng được, không cần guard đăng ký thật': với /reading đúng là không lưu, nhưng phiên ghép cặp thì được lưu trong palm_match_sessions; còn guest chỉ là user có JWT (JwtAuthGuard global), controller không có @Public hay phân biệt guest.
- Doc chưa nhắc 2 config thật của luồng ghép cặp: PALM_MATCH_QUEUE_MAX_WAIT_SECONDS và PALM_MATCH_SESSION_DURATION_SECONDS (chỉ có PALM_MATCH_TARGET_NAME_MAX_LENGTH).
- Không có job nền: hạn chờ queue và hạn phiên được xử lý lazy (expireSession/expireActiveForUser) khi user gọi joinQueue/getCurrent/flip/rate, nên phiên bị bỏ rơi chỉ hết hạn khi một trong 2 người quay lại.
- Palm Match không publish event realtime nào (dossier: không có socket-emit); client phát hiện được ghép cặp hoàn toàn bằng REST polling GET /palm-match/current.
- Chỉ Safety.canPair được kiểm tra lúc ghép; flip/rating không kiểm tra lại block, nên block xảy ra sau khi đã ghép không chặn phiên đang chạy (suy luận từ code, không thấy check ở flip/rate).
