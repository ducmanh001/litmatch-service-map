# Xem phim chung ẩn danh: hàng đợi, đồng bộ playback, rating

id: flow:movie-match-anonymous
group: Ghép đôi & gọi
services: module:movie-match, module:safety, module:friend, module:signaling-gateway

Người lạ xếp hàng xem phim chung. movie-match ghép cặp (hỏi safety có được ghép không), đồng bộ playback qua realtime tới signaling-gateway, và chỉ khi cả hai like thì friend mới tạo Friendship.

Ba pha của một phiên xem chung ẩn danh: ghép hàng đợi, đồng bộ playback realtime, rating dẫn tới Friendship khi mutual-like.

## Participants

- web (client): Web client
- movie-match (service): movie-match
- safety (service): safety
- friend (service): friend
- postgres (infra): PostgreSQL
- redis (infra): Redis pub/sub
- signaling-gateway (service): signaling-gateway

## Steps

1. web -> movie-match [call/http]: POST /movie-match/anon/queue: vào hàng đợi ghép người lạ (Mọi route REST đều qua JwtAuthGuard trừ route @Public.)
2. movie-match -> postgres [call/shared-db]: Mở transaction, khóa advisory của matcher, xử lý hết hạn, enqueue và đọc ứng viên {apps/core-api/src/modules/movie-match/movie-match.service.ts:295, apps/core-api/src/modules/movie-match/movie-match.service.ts:680}
3. movie-match -> safety [call/di-call]: canPair(userId, ứng viên): kiểm tra block/report đúng lúc ghép (Gọi trong vòng lặp ứng viên, nằm trong transaction của movie-match.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:338}
4. movie-match -> postgres [call/shared-db]: Tạo session, xóa khỏi hàng đợi, ghi active participant cho cả hai {apps/core-api/src/modules/movie-match/movie-match.service.ts:710, apps/core-api/src/modules/movie-match/movie-match.service.ts:355}
5. movie-match -> web [return/http]: Trả trạng thái queued hoặc phiên đã ghép (server chọn video)
6. web -> movie-match [call/http]: PATCH /anon/sessions/:id/state: gửi vị trí và play/pause
7. movie-match -> postgres [call/shared-db]: Khóa session, lazy-expire, lưu playback state mới {apps/core-api/src/modules/movie-match/movie-match.service.ts:421, apps/core-api/src/modules/movie-match/movie-match.service.ts:437}
8. movie-match -> redis [async/redis-pubsub]: Publish MovieStateChanged tới kênh realtime của cả hai người (publishAnonPlayback được gọi ngay trong callback transaction, sau manager.save và TRƯỚC khi commit (không phải sau commit); publishRealtimeEvent bắt lỗi và chỉ log cảnh báo, client còn polling fallback.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:856, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
9. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
10. signaling-gateway -> web [async/socket]: Relay playback state xuống socket room user:{userId} của cả hai {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
11. web -> movie-match [call/http]: POST /anon/sessions/:id/rating: gửi rating (like/boring/rude)
12. movie-match -> friend [call/di-call]: Chỉ khi cả hai like: ensureFriendship(manager, low, high, MovieMatch) (Truyền cùng manager nên Friendship nằm trong transaction của rating.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:510}
13. movie-match -> postgres [call/shared-db]: Lưu rating; chốt session Matched (cả hai like) hoặc NotMatched (rating khác like) (Nếu mới một bên like và bên kia chưa rate thì chỉ lưu rating, session vẫn Active.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:519}
14. movie-match -> web [return/http]: Trả anonView với trạng thái và rating mới

## Invariants

- Advisory lock được lấy ngay đầu transaction ghép hàng đợi, nên các lần ghép đồng thời bị tuần tự hóa trước khi đọc ứng viên.
- canPair của safety được gọi lại ngay lúc ghép, trong cùng transaction, nên không dựa vào trạng thái block/report cũ từ lúc enqueue.
- friend.ensureFriendship nhận cùng manager của movie-match và chạy trước manager.save(session) cuối cùng, nên Friendship và việc chốt session nằm trong một transaction.
- Chỉ khi cả lowRating và highRating là Like mới gọi ensureFriendship; rating khác Like kết thúc phiên NotMatched mà không mở Friendship.

## Failure modes

- when Publish Redis realtime lỗi khi cập nhật playback: Lỗi bị bắt và chỉ log cảnh báo; transaction vẫn tiếp tục, client dựa vào polling fallback.
- when Phiên đã hết phần xem (status không Active hoặc watchEndedAt khác null) khi PATCH state: Ném DomainException ENDED (HTTP 409) và transaction rollback, không lưu và không publish.
- when Rating khác với rating đã chốt trước đó: Ném RATING_CONFLICT (409), không thay đổi session.
- when Gửi rating khi phần xem chung chưa kết thúc: Ném RATING_NOT_OPEN (409), không chạm Friendship.

## Endpoints covered

- POST /movie-match/anon/queue [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- PATCH /movie-match/anon/sessions/:id/state [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- POST /movie-match/anon/sessions/:id/rating [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
