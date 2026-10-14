# Xem phim ẩn danh: chat, reaction và kết thúc

id: flow:movie-match-anon-chat-finish-reactions
group: Ghép đôi & gọi
services: module:movie-match, module:signaling-gateway

Trong phiên xem phim ẩn danh, hai người chat có Idempotency-Key, gửi reaction tạm thời, bấm kết thúc và dọn trạng thái. movie-match lưu dữ liệu vào PostgreSQL còn Redis và signaling-gateway đẩy sự kiện realtime xuống cả hai client.

Chat idempotent qua DB, reaction và đổi trạng thái đi qua Redis tới gateway, kết thúc và dọn phiên trong transaction.

## Participants

- web (client): Web client
- movie-match (service): movie-match
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): signaling-gateway

## Steps

1. web -> movie-match [call/http]: POST /anon/sessions/:id/messages kèm header Idempotency-Key
2. movie-match -> postgres [call/shared-db]: Kiểm tra người tham gia rồi lưu message với key đã gắn tiền tố user (Nếu vi phạm unique: cùng nội dung và session thì trả lại message cũ (replay), khác nội dung thì lỗi 409 MESSAGE_IDEMPOTENCY_CONFLICT.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:553}
3. movie-match -> web [return/http]: Trả message đã lưu (hoặc bản replay)
4. web -> movie-match [call/http]: GET /anon/sessions/:id/messages phân trang theo cursor seq
5. movie-match -> postgres [call/shared-db]: Kiểm tra người tham gia, đọc message có seq lớn hơn cursor {apps/core-api/src/modules/movie-match/movie-match.service.ts:600}
6. web -> movie-match [call/http]: POST /anon/sessions/:id/reactions (emoji trong whitelist, throttle 60/phút)
7. movie-match -> postgres [call/shared-db]: Đọc session, xác nhận người gửi thuộc phiên ẩn danh đang Active {apps/core-api/src/modules/movie-match/movie-match.service.ts:817}
8. movie-match -> redis [async/redis-pubsub]: Publish MovieReactionSent cho cả hai user, không lưu DB (publishRealtimeEvent bắt lỗi publish và chỉ logger.warn, nên reaction không làm request lỗi.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:637}
9. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
10. signaling-gateway -> web [async/socket]: Đẩy movie reaction xuống socket room của từng user {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
11. web -> movie-match [call/http]: POST /anon/sessions/:id/finish kết thúc phần xem, sang phase rating
12. movie-match -> postgres [call/shared-db]: Khóa session, tự hết hạn nếu quá giờ, lưu watchEndedAt và dừng playback (Idempotent: nếu watchEndedAt đã có (hoặc phiên vừa bị lazy-expire sang Ended) thì không ghi và không publish lại.) {apps/core-api/src/modules/movie-match/movie-match.service.ts:458}
13. movie-match -> redis [async/redis-pubsub]: Publish MovieStateChanged (isPlaying=false) cho cả hai user (Publish nằm trong callback transaction, lỗi publish bị nuốt (chỉ log).) {apps/core-api/src/modules/movie-match/movie-match.service.ts:856}
14. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
15. signaling-gateway -> web [async/socket]: Đẩy trạng thái playback đã dừng xuống cả hai client {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
16. web -> movie-match [call/http]: GET /anon/current để phục hồi trạng thái, hết hạn lazy
17. movie-match -> postgres [call/shared-db]: Trong 1 transaction: lazy-expire, đọc phiên active hoặc queue entry {apps/core-api/src/modules/movie-match/movie-match.service.ts:366}
18. web -> movie-match [call/http]: DELETE /anon/current huỷ queue hoặc đóng phiên cho cả hai
19. movie-match -> postgres [call/shared-db]: Advisory lock, xoá queue, đặt phiên Ended/Cancelled, xoá active participant {apps/core-api/src/modules/movie-match/movie-match.service.ts:407}
20. movie-match -> web [return/http]: Trả state Idle

## Invariants

- Idempotency-Key được gắn tiền tố userId rồi lưu cùng message với ràng buộc unique, nên retry cùng nội dung trả lại message cũ thay vì tạo bản trùng.
- Reaction chỉ là realtime: service không ghi DB, chỉ kiểm tra whitelist và tư cách người tham gia rồi publish qua Redis, gateway mới là nơi đẩy xuống socket.
- endAnonWatch khóa dòng session (pessimistic_write) trước khi đổi watchEndedAt, nên hai người bấm kết thúc cùng lúc chỉ ghi và publish một lần.
- dismissAnonCurrent lấy advisory lock rồi đóng phiên cho cả hai bên, để người kia không phải chờ vô hạn.

## Failure modes

- when Redis publish thất bại khi gửi reaction hoặc đổi playback: publishRealtimeEvent chỉ log cảnh báo, request vẫn thành công; client dựa vào polling GET /anon/current.
- when Gửi message khi phiên không còn Active: Trả lỗi 409 MovieMatchErrors.ENDED.
- when Idempotency-Key trùng nhưng nội dung hoặc session khác: Trả 409 MESSAGE_IDEMPOTENCY_CONFLICT.
- when Reaction ngoài whitelist: Trả 422 REACTION_INVALID, không publish.
- when Kết thúc phần xem: publish MovieStateChanged chạy trong callback transaction, trước khi commit: Suy luận: event có thể tới client trước commit; publish lỗi chỉ bị log, nhưng nếu commit sau đó lỗi thì event đã phát đi dù DB không đổi.

## Endpoints covered

- GET /movie-match/anon/current [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- DELETE /movie-match/anon/current [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- POST /movie-match/anon/sessions/:id/finish [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- POST /movie-match/anon/sessions/:id/messages [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- GET /movie-match/anon/sessions/:id/messages [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
- POST /movie-match/anon/sessions/:id/reactions [module:movie-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/movie-match.md
