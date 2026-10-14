# Phòng chat ẩn danh Soul Match và mutual-like

id: flow:soul-match-session
group: Ghép đôi & gọi
services: module:soul-match, module:matching, module:friend, module:signaling-gateway

Hai người đã được Matching ghép đôi chat ẩn danh rồi đánh giá lẫn nhau; khi cả hai like thì Soul Match phải nhờ Friend mở Friendship và nhập lịch sử chat, đồng thời khoá hàng MatchSession của Matching để hai rater không bỏ lỡ nhau.

Luồng POST /soul-match/sessions/:id/rating: kiểm tra phòng, khoá session, ghi rating, mở Friendship trong cùng transaction rồi đẩy sự kiện realtime; và POST /soul-match/sessions/:id/end: Matching kết thúc session trong transaction rồi Soul Match đẩy soul.ended cho cả hai.

## Participants

- web (client): Web client
- soul-match (service): Soul Match
- matching (service): Matching
- friend (service): Friend
- postgres (infra): PostgreSQL
- redis (infra): Redis pub/sub
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> soul-match [call/http]: POST /soul-match/sessions/:id/rating gửi verdict rude|boring|like
2. soul-match -> matching [call/di-call]: Lấy MatchSession để xác định phòng và pha hiện tại {apps/core-api/src/modules/soul-match/soul-match.service.ts:434}
3. soul-match -> friend [call/di-call]: Kiểm tra hai người đã là bạn chưa (matched) {apps/core-api/src/modules/soul-match/soul-match.service.ts:474}
4. soul-match -> postgres [call/shared-db]: Mở transaction, khoá pessimistic_write hàng match_sessions (chỉ khoá, không ghi) (Bảng match_sessions thuộc Matching; Soul Match chỉ khoá để tuần tự hoá hai rater.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:311}
5. soul-match -> postgres [call/shared-db]: Đọc rating cũ rồi lưu SoulMatchRating của người đánh giá {apps/core-api/src/modules/soul-match/soul-match.service.ts:331}
6. soul-match -> postgres [call/shared-db]: Nếu là like: đọc rating của đối phương và toàn bộ tin nhắn ẩn danh {apps/core-api/src/modules/soul-match/soul-match.service.ts:345}
7. soul-match -> friend [call/di-call]: Cả hai like: ensureFriendship(manager, ...) kèm lịch sử chat làm seed (EntityManager của transaction được truyền sang Friend.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:362}
8. friend -> soul-match [return/di-call]: Trả created (Friendship mới hay đã có) rồi transaction commit
9. soul-match -> redis [async/redis-pubsub]: Sau commit, nếu Friendship mới: publish soul.matched cho cả hai user {apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11, apps/core-api/src/modules/soul-match/soul-match.service.ts:398}
10. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
11. signaling-gateway -> web [async/socket]: Relay envelope xuống socket trong room user:{userId} {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
12. soul-match -> web [return/http]: Trả RateResult { verdict, matched }
13. web -> soul-match [call/http]: POST /soul-match/sessions/:id/end rời phòng Soul Match (Route trả 204 No Content; chỉ người trong phiên mới đóng được vì Matching kiểm lại thành viên.) {apps/core-api/src/modules/soul-match/soul-match.controller.ts:125}
14. soul-match -> matching [call/di-call]: Nhờ Matching kết thúc MatchSession Soul của người gọi (Soul Match không tự ghi match_sessions: bảng này thuộc Matching, nên endSession gọi endSoulSession rồi mới publish.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:105}
15. matching -> postgres [call/shared-db]: Trong transaction: khoá hàng session, đặt Ended và endedAt (Phiên đã Ended thì trả lại nguyên trạng; không thuộc phiên, sai loại Soul hoặc không ở Confirmed thì ném lỗi 404/409 và không có gì được publish.) {apps/core-api/src/modules/matching/matching.service.ts:568, apps/core-api/src/modules/matching/matching.service.ts:576}
16. soul-match -> redis [async/redis-pubsub]: Sau commit: publish soul.ended (reason member_left) cho cả hai user (Realtime chỉ là tín hiệu để UI đóng ngay; việc kết thúc đã bền trong DB trước đó. Publish chạy cho cả hai user qua Promise.all, kể cả khi phiên đã Ended từ trước.) {apps/core-api/src/modules/soul-match/soul-match.service.ts:107, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
17. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
18. signaling-gateway -> web [async/socket]: Relay soul.ended xuống socket room user:{userId} của hai bên {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
19. soul-match -> web [return/http]: Trả 204 No Content

## Invariants

- Soul Match khoá hàng match_sessions (FOR UPDATE) trước khi ghi rating để hai rater chạy song song nhìn thấy nhau; không khoá thì hai like có thể cùng kết luận chưa đủ hai like.
- Kiểm tra mutual-like và ensureFriendship nằm CÙNG transaction với insert rating, nên không có trạng thái rating đã ghi mà Friendship chưa mở khi cả hai like.
- Matching là nơi duy nhất ghi match_sessions; Soul Match chỉ khoá hàng này, còn Friendship và lịch sử chat do Friend ghi qua ensureFriendship.
- Realtime chỉ được publish sau khi transaction commit và chỉ khi newlyMatched, nên replay rating không bắn lại soul.matched.
- Rời phòng là transition bền do Matching ghi trong transaction có khoá; soul.ended chỉ là tín hiệu đóng UI, được publish sau khi Matching đã commit.

## Failure modes

- when Publish Redis soul.matched thất bại: publishRealtimeEvent bắt lỗi và chỉ ghi logger.warn; Friendship đã commit, client dựa vào polling fallback.
- when Hai lần submit song song cùng một user vi phạm unique (session, rater): Nhờ khoá pessimistic_write trên match_sessions, lần gửi thứ hai thường thấy rating đã có và replay/assertSameVerdict; nếu vẫn dính unique (session, rater) thì catch isUniqueViolation đọc lại rating ngoài transaction: cùng verdict trả kết quả replay, khác verdict bị assertSameVerdict từ chối.
- when Lỗi trong transaction (ví dụ ensureFriendship ném lỗi): Transaction rollback, rating không được lưu và không có sự kiện realtime nào.
- when Người dùng gửi rating khi phòng đã Closed: Ném RATING_NOT_OPEN (409) trước khi mở transaction.
- when Người gọi không thuộc phiên, phiên không phải Soul hoặc chưa ở trạng thái Confirmed khi gọi end: Matching ném DomainException (404 SESSION_NOT_FOUND hoặc 409 SESSION_NOT_CALLABLE / SESSION_NOT_PENDING), transaction không ghi gì và không publish soul.ended.
- when Gọi end lần nữa khi phiên đã Ended: endSoulSession trả lại session nguyên trạng nhưng endSession không có nhánh chặn, nên soul.ended được publish lại cho cả hai user.

## Endpoints covered

- POST /soul-match/sessions/:id/rating [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
- POST /soul-match/sessions/:id/end [module:soul-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/soul-match.md
