# Gửi tin nhắn ẩn danh trong phòng Soul Match

id: flow:soul-match-send-message
group: Ghép đôi & gọi
services: module:soul-match, module:matching, module:friend, module:user, module:signaling-gateway

Thành viên gửi tin nhắn ẩn danh trong phòng Soul Match. Soul Match không sở hữu MatchSession và quan hệ bạn bè, nên phải hỏi Matching và Friend, hỏi User về trạng thái ban, rồi lưu tin trong transaction và đẩy realtime qua Redis và gateway.

Soul Match kiểm quyền với Matching, Friend, User, lưu tin trong transaction rồi đẩy realtime tới cả hai phía qua Redis và signaling-gateway.

## Participants

- web (client): Web client
- soul-match (service): Soul Match
- matching (service): Matching
- friend (service): Friend
- user (service): User
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling Gateway

## Steps

1. web -> soul-match [call/http]: POST /soul-match/sessions/:id/messages kèm Idempotency-Key (Route được JwtAuthGuard bảo vệ; riêng endpoint này throttle 60 request/phút.) {apps/core-api/src/modules/soul-match/soul-match.controller.ts:85}
2. soul-match -> matching [call/di-call]: Lấy MatchSession theo id để xác nhận người gọi là thành viên (Không phải thành viên hoặc không có session thì trả 404 gộp; session chưa confirmed thì CHAT_NOT_OPEN.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:434}
3. soul-match -> friend [call/di-call]: Hỏi hai người đã là bạn chưa (matched) để tính phase phòng (Nếu đã là bạn hoặc còn trong thời gian chat thì phase là Chatting; ngược lại Rating hoặc Closed.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:474}
4. soul-match -> user [call/di-call]: Lấy hồ sơ người gửi để kiểm tra tài khoản có bị khoá không (Status Banned thì trả 403 USER_BANNED.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:137}
5. soul-match -> postgres [call/shared-db]: Khoá dòng MatchSession (pessimistic_write) và kiểm tra lại giờ chat (Cùng lock với bước rating để tin nhắn không lọt giữa snapshot và chuyển sang Friend Chat.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:163}
6. soul-match -> postgres [call/shared-db]: Lưu SoulChatMessage với idempotencyKey có tiền tố userId (Trùng khoá thì unique violation, được xử lý như replay.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:186}
7. soul-match -> redis [async/redis-pubsub]: Publish soul message tới realtime:user của người gửi (me) và đối phương (partner) (senderRole tính riêng cho từng người nhận để gateway không cần biết về ẩn danh. Lỗi publish bị logger.warn nuốt.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:215, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
8. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận qua PSUBSCRIBE realtime:user:* {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
9. signaling-gateway -> web [async/socket]: Emit sự kiện tới room user của từng người nhận {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
10. soul-match -> web [return/http]: Trả SoulMessageDto của tin vừa lưu (hoặc tin cũ nếu replay)

## Invariants

- Phải hỏi Matching và Friend trước khi mở transaction: phase phòng (chat, rating, đóng) suy ra từ MatchSession và việc đã là bạn, mà Soul Match không sở hữu hai dữ liệu này.
- Lock dòng MatchSession trong cùng transaction với lúc lưu tin, dùng chung lock với rating, để không có tin nhắn nào lọt giữa bước snapshot và bước chuyển sang Friend Chat.
- Publish realtime chỉ chạy sau khi transaction commit xong, nên người nhận không bao giờ thấy sự kiện của tin chưa được lưu.

## Failure modes

- when Publish Redis thất bại: Lỗi bị nuốt và ghi logger.warn; tin đã lưu, request vẫn thành công, client dùng polling fallback.
- when Idempotency-Key bị gửi lại (unique violation): Cùng session và nội dung thì trả lại tin cũ, không bắn realtime lại; khác nội dung thì 409 MESSAGE_IDEMPOTENCY_CONFLICT.
- when Hết giờ chat và chưa là bạn, hoặc tài khoản bị khoá: Trả 409 CHAT_NOT_OPEN hoặc 403 USER_BANNED, không ghi gì vào DB.
- when Nội dung vượt SOUL_CHAT_MESSAGE_MAX_LENGTH: Trả 422 MESSAGE_TOO_LONG trước khi mở transaction, không ghi DB.

## Endpoints covered

- POST /soul-match/sessions/:id/messages [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
