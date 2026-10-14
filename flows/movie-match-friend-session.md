# Xem phim chung giữa bạn bè

id: flow:movie-match-friend-session
group: Ghép đôi & gọi
services: module:movie-match, module:friend, module:signaling-gateway

Hai người bạn xem chung một video YouTube: movie-match phải kiểm tra quan hệ bạn bè với friend, ghi phiên và 2 dòng participant vào PostgreSQL, rồi đẩy sự kiện qua Redis và signaling-gateway để người còn lại nhận playback state tức thì.

Tạo phiên sau kiểm tra bạn bè, đồng bộ playback state và kết thúc phiên, mỗi thay đổi đều được đẩy realtime tới cả hai người.

## Participants

- web (client): Web client
- movie-match (service): movie-match
- friend (service): friend
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): signaling-gateway

## Steps

1. web -> movie-match [call/http]: POST /movie-match/sessions với friendUserId và videoUrl YouTube (Route cần JWT (JwtAuthGuard toàn cục). videoUrl được kiểm whitelist domain bằng new URL() trước khi làm gì khác.)
2. movie-match -> friend [call/di-call]: Kiểm tra hai người có phải bạn bè không (Nếu không phải bạn thì ném NOT_FRIEND (404).) {apps/core-api/src/modules/movie-match/movie-match.service.ts:123}
3. movie-match -> postgres [call/shared-db]: Tìm phiên active của cặp bạn (idempotent replay) {apps/core-api/src/modules/movie-match/movie-match.service.ts:650}
4. movie-match -> postgres [call/shared-db]: Transaction: lưu MovieSession và insert 2 MovieSessionActiveParticipant (Unique violation (user đã active cặp khác) làm rollback cả transaction rồi đọc lại để phân biệt replay với 409.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:139}
5. movie-match -> redis [async/redis-pubsub]: Publish MovieSessionStarted cho từng user trong cặp (Chỉ publish khi phiên vừa được tạo mới; nếu đã có phiên active của cặp (replay, bước 3 hoặc nhánh race) thì trả phiên cũ và không publish lại.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:182}
6. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
7. signaling-gateway -> web [async/socket]: Emit movie session started tới room của từng user {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
8. movie-match -> web [return/http]: Trả MovieSessionDto của phiên vừa tạo
9. web -> movie-match [call/http]: PATCH /movie-match/sessions/:id/state với positionSeconds và isPlaying (Có GET /movie-match/sessions/:id làm poll fallback: chỉ đọc MovieSession và kiểm caller là participant.)
10. movie-match -> postgres [call/shared-db]: Kiểm participant, UPDATE playback state (last-write-wins) (Không dùng transaction hay lock, phiên đã ended thì trả 409.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:209}
11. movie-match -> redis [async/redis-pubsub]: Publish MovieStateChanged cho cả hai user {apps/core-api/src/modules/movie-match/movie-match.service.ts:232}
12. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận envelope từ realtime:user:* {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
13. signaling-gateway -> web [async/socket]: Emit playback state mới để người kia đồng bộ video {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
14. web -> movie-match [call/http]: POST /movie-match/sessions/:id/end để kết thúc phiên
15. movie-match -> postgres [call/shared-db]: Transaction: đặt status Ended và xoá 2 dòng active participant (Gọi lại trên phiên đã ended là no-op idempotent.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:249}
16. movie-match -> redis [async/redis-pubsub]: Publish MovieSessionEnded cho cả hai user {apps/core-api/src/modules/movie-match/movie-match.service.ts:276}
17. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận envelope kết thúc phiên {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
18. signaling-gateway -> web [async/socket]: Emit sự kiện kết thúc tới cả hai client {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
19. movie-match -> web [return/http]: Trả MovieSessionDto trạng thái ended

## Invariants

- movie-match phải hỏi friend.areFriends trước khi ghi gì, vì Friendship thuộc friend và movie-match không được tự suy ra quan hệ bạn bè.
- MovieSession và 2 dòng MovieSessionActiveParticipant phải nằm trong một transaction để vi phạm unique của một user rollback cả phiên, không để lại row mồ côi.
- Publish realtime chỉ chạy sau khi ghi DB xong, nên sự kiện không bao giờ báo trạng thái chưa được lưu; DB là nguồn sự thật, realtime chỉ là thông báo.

## Failure modes

- when Publish Redis thất bại: publishRealtimeEvent bắt lỗi và chỉ logger.warn, request vẫn thành công; client dùng GET /movie-match/sessions/:id làm poll fallback.
- when Hai người không phải bạn bè: createSession ném NOT_FRIEND (404), không ghi DB.
- when Một trong hai user đã active phiên với người khác: Unique violation làm rollback transaction, service đọc lại cặp và trả 409 ALREADY_ACTIVE nếu không phải replay.
- when Cập nhật state trên phiên đã ended: updateState ném ENDED (409), không UPDATE và không publish.

## Endpoints covered

- POST /movie-match/sessions [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- PATCH /movie-match/sessions/:id/state [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- POST /movie-match/sessions/:id/end [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- GET /movie-match/sessions/:id [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
