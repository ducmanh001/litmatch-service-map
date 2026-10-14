# Đọc phòng Soul Match và lộ hồ sơ sau mutual-like

id: flow:soul-match-room-reads-and-reveal
group: Ghép đôi & gọi
services: module:soul-match, module:matching, module:friend, module:user

Phòng chat ẩn danh Soul Match thuộc soul-match nhưng MatchSession do matching sở hữu và Friendship do friend sở hữu, nên mọi lần đọc phòng, tin nhắn hay hồ sơ đối phương đều phải hỏi hai service kia và suy ra phase theo giờ server.

Ba endpoint đọc đều đi qua getRoomForMember (matching + friend); hồ sơ đối phương chỉ lộ khi Friendship tồn tại.

## Participants

- web (client): Web client
- soul-match (service): Soul Match
- matching (service): Matching
- friend (service): Friend
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> soul-match [call/http]: GET /soul-match/sessions/:id (poll trạng thái phòng)
2. soul-match -> matching [call/di-call]: getRoomForMember: tìm MatchSession theo id, kiểm tra là thành viên (Không phải thành viên hoặc không có session thì gộp thành 404 SESSION_NOT_FOUND; không phải session soul đã đủ 2 confirm thì 409 CHAT_NOT_OPEN.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:434}
3. matching -> soul-match [return/di-call]: Trả MatchSession (userA/B, status, confirmedAAt/BAt)
4. soul-match -> friend [call/di-call]: areFriends(userA, userB) để lấy cờ matched (phase = Chatting nếu matched hoặc chưa hết SOUL_CHAT_DURATION_SECONDS; sau đó Rating, rồi Closed; session Ended thì luôn Closed.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:474}
5. friend -> soul-match [return/di-call]: Trả matched (đã là bạn hay chưa)
6. soul-match -> postgres [call/shared-db]: Đọc rating của chính mình (myVerdict) {apps/core-api/src/modules/soul-match/soul-match.service.ts:92}
7. soul-match -> web [return/http]: Trả room + phase + myVerdict + matched
8. web -> soul-match [call/http]: GET /soul-match/sessions/:id/messages (cursor)
9. soul-match -> matching [call/di-call]: getRoomForMember: tìm MatchSession theo id, kiểm tra là thành viên (Không phải thành viên hoặc không có session thì gộp thành 404 SESSION_NOT_FOUND; không phải session soul đã đủ 2 confirm thì 409 CHAT_NOT_OPEN.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:434}
10. soul-match -> friend [call/di-call]: areFriends(userA, userB) để lấy cờ matched (phase = Chatting nếu matched hoặc chưa hết SOUL_CHAT_DURATION_SECONDS; sau đó Rating, rồi Closed; session Ended thì luôn Closed.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:474}
11. soul-match -> postgres [call/shared-db]: Đọc tin nhắn seq > cursor, sắp xếp tăng dần (Nếu phase = Closed thì ném 409 CHAT_NOT_OPEN trước khi truy vấn.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:275}
12. soul-match -> web [return/http]: Trả trang tin nhắn ẩn danh
13. web -> soul-match [call/http]: GET /soul-match/sessions/:id/partner (xem hồ sơ đối phương)
14. soul-match -> matching [call/di-call]: getRoomForMember: tìm MatchSession theo id, kiểm tra là thành viên (Không phải thành viên hoặc không có session thì gộp thành 404 SESSION_NOT_FOUND; không phải session soul đã đủ 2 confirm thì 409 CHAT_NOT_OPEN.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:434}
15. soul-match -> friend [call/di-call]: areFriends(userA, userB) để lấy cờ matched (phase = Chatting nếu matched hoặc chưa hết SOUL_CHAT_DURATION_SECONDS; sau đó Rating, rồi Closed; session Ended thì luôn Closed.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:474}
16. soul-match -> friend [call/di-call]: areFriends(user, partner): kiểm tra lần nữa trước khi lộ hồ sơ (Chưa là bạn thì 403 PARTNER_LOCKED, không gọi user.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:412}
17. soul-match -> user [call/di-call]: getByIdOrThrow(partnerId) dựng PublicProfileDto {apps/core-api/src/modules/soul-match/soul-match.service.ts:420}
18. soul-match -> web [return/http]: Trả PublicProfileDto của đối phương

## Invariants

- MatchSession do matching ghi và Friendship do friend ghi, nên soul-match chỉ đọc qua MatchingService.findSessionById và FriendService.areFriends, không tự giữ bản sao trạng thái.
- Friendship là nguồn sự thật để mở khóa hồ sơ: getPartnerProfile gọi areFriends lần nữa rồi mới lấy PublicProfileDto từ user, nên không thể đọc hồ sơ bằng cách đoán sessionId.
- Không phải thành viên và không có session cùng trả 404 để không thành oracle dò sessionId; phase luôn tính theo giờ server từ mốc confirmedAAt/confirmedBAt muộn hơn (chatEndsAt = mốc + SOUL_CHAT_DURATION_SECONDS), không tin client.

## Failure modes

- when Session không tồn tại hoặc caller không phải thành viên: 404 SESSION_NOT_FOUND
- when Session không phải soul hoặc chưa Confirmed/Ended: 409 CHAT_NOT_OPEN
- when Đọc tin nhắn khi phase = Closed: 409 CHAT_NOT_OPEN, lịch sử ẩn danh không đọc lại được
- when Cursor không hợp lệ: 400 CURSOR_INVALID
- when Gọi /partner khi chưa là bạn: 403 PARTNER_LOCKED

## Endpoints covered

- GET /soul-match/sessions/:id [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
- GET /soul-match/sessions/:id/messages [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
- GET /soul-match/sessions/:id/partner [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
