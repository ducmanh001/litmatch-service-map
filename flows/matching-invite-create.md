# Mời chơi có chủ đích: tạo và xem lời mời

id: flow:matching-invite-create
group: Ghép đôi & gọi
services: module:matching, module:safety, module:user, module:notification

Người dùng mời trực tiếp một người khác chơi Voice/Soul Match. Module matching sở hữu MatchInvite nhưng phải hỏi Safety (hidden-set), User (trạng thái invitee), ghi thông báo và push qua Notification, nên nhiều service cùng tham gia; hộp thư và chi tiết lời mời cũng kiểm lại hidden-set và đọc hồ sơ người mời.

Luồng tạo lời mời (throttle, rate limit Redis, kiểm hidden-set, lưu, thông báo + push) rồi xem hộp thư và chi tiết lời mời.

## Participants

- web (client): Web client
- matching (service): Matching
- safety (service): Safety
- user (service): User
- notification (service): Notification
- redis (infra): Redis
- postgres (infra): PostgreSQL

## Steps

1. web -> matching [call/http]: POST /matching/invites (inviteeUserId, matchType) (Route yêu cầu JWT (APP_GUARD) và bị Throttle 10 request/phút.) {apps/core-api/src/modules/matching/controllers/invite.controller.ts:53}
2. matching -> redis [call/redis-state]: Tiêu thụ rate limit lời mời theo inviter mỗi giờ (Nếu vượt MATCHING_INVITE_RATE_LIMIT_PER_HOUR thì ném INVITE_RATE_LIMITED (429).) {apps/core-api/src/modules/matching/services/invite.service.ts:109}
3. matching -> safety [call/di-call]: Lấy hidden-set của inviter, loại invitee bị ẩn/block (Trùng hidden-set thì trả cùng mã lỗi INVITE_TARGET_UNAVAILABLE (422).) {apps/core-api/src/modules/matching/services/invite.service.ts:581}
4. matching -> user [call/di-call]: Đọc invitee, yêu cầu tồn tại và status Active (Không tồn tại hoặc không Active cũng trả INVITE_TARGET_UNAVAILABLE.) {apps/core-api/src/modules/matching/services/invite.service.ts:589}
5. matching -> postgres [call/shared-db]: Lưu MatchInvite trạng thái Pending kèm expiresAt (Vi phạm unique (đã có lời mời Pending tới đúng người này) thì trả INVITE_ALREADY_PENDING (409).) {apps/core-api/src/modules/matching/services/invite.service.ts:131}
6. matching -> notification [call/di-call]: Ghi thông báo MatchInviteReceived cho invitee (createInvite không mở transaction; thông báo được ghi sau khi invite đã lưu.) {apps/core-api/src/modules/matching/services/invite.service.ts:153}
7. matching -> notification [async/di-call]: Đẩy push tới invitee (best-effort) (sendPush bắt mọi lỗi của push provider và web push, chỉ ghi log cảnh báo.) {apps/core-api/src/modules/matching/services/invite.service.ts:162}
8. matching -> user [call/di-call]: Đọc hồ sơ inviter để dựng MatchInviteDto {apps/core-api/src/modules/matching/controllers/invite.controller.ts:138}
9. matching -> web [return/http]: 201 MatchInviteDto của lời mời vừa tạo
10. web -> matching [call/http]: GET /matching/invites (hộp thư lời mời đang chờ)
11. matching -> safety [call/di-call]: Lấy lại hidden-set của người nhận mỗi lần đọc hộp thư (Block/report có thể phát sinh sau khi invite được tạo nên phải kiểm lại.) {apps/core-api/src/modules/matching/services/invite.service.ts:199}
12. matching -> postgres [call/shared-db]: Truy vấn invite Pending của invitee, loại inviter bị ẩn {apps/core-api/src/modules/matching/services/invite.service.ts:202}
13. matching -> user [call/di-call]: Đọc hàng loạt hồ sơ các inviter của trang kết quả {apps/core-api/src/modules/matching/controllers/invite.controller.ts:145}
14. matching -> web [return/http]: 200 trang lời mời kèm thông tin người mời
15. web -> matching [call/http]: GET /matching/invites/:id (chi tiết một lời mời)
16. matching -> postgres [call/shared-db]: Đọc MatchInvite theo id, chỉ inviter/invitee được xem (Người ngoài cuộc nhận INVITE_FORBIDDEN (403), không tồn tại nhận INVITE_NOT_FOUND (404).) {apps/core-api/src/modules/matching/services/invite.service.ts:171}
17. matching -> user [call/di-call]: Đọc hồ sơ inviter để dựng MatchInviteDto {apps/core-api/src/modules/matching/controllers/invite.controller.ts:138}
18. matching -> web [return/http]: 200 MatchInviteDto của lời mời

## Invariants

- Rate limit Redis và Throttle chạy trước mọi truy vấn Safety/User/DB để chặn spam lời mời rẻ nhất trước.
- Safety và User được hỏi trước khi lưu MatchInvite, và mọi lý do từ chối dùng cùng mã INVITE_TARGET_UNAVAILABLE để người gọi không dò được ai đã block mình.
- Thông báo và push chỉ chạy sau khi MatchInvite đã lưu, nên invitee không bao giờ nhận thông báo về lời mời không tồn tại.
- Hộp thư kiểm lại hidden-set ở mỗi lần đọc vì block/report có thể phát sinh sau khi invite được tạo.

## Failure modes

- when Inviter vượt giới hạn lời mời mỗi giờ: Ném INVITE_RATE_LIMITED (429), không lưu invite, không gửi thông báo.
- when Invitee nằm trong hidden-set, không tồn tại hoặc không Active: Ném INVITE_TARGET_UNAVAILABLE (422) với cùng thông điệp cho mọi lý do.
- when Đã có lời mời Pending tới đúng người này (vi phạm unique): Ném INVITE_ALREADY_PENDING (409), không tạo thông báo.
- when Push provider hoặc web push lỗi: sendPush nuốt lỗi và ghi log; invite và thông báo in-app vẫn còn, request vẫn thành công.
- when Người không phải inviter/invitee xem chi tiết lời mời: Ném INVITE_FORBIDDEN (403).

## Endpoints covered

- POST /matching/invites [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- GET /matching/invites [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- GET /matching/invites/:id [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
