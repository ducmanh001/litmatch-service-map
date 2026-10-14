# Mini game giữa hai bạn bè

id: flow:mini-game-session
group: Ghép đôi & gọi
services: module:mini-game, module:friend, module:signaling-gateway

Một bạn mời bạn bè chơi oẳn tù tì, hai bên nộp move ẩn theo lượt; mini-game phải hỏi friend xem hai người có là bạn không, ghi phiên vào PostgreSQL rồi báo realtime qua Redis và signaling-gateway tới web.

Tạo phiên (kiểm tra bạn bè, ghi DB, báo realtime) rồi nộp move; khi đủ 2 move thì server chốt thắng thua và báo cả hai bên.

## Participants

- web (client): Web client
- mini-game (service): Mini Game
- friend (service): Friend
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling Gateway

## Steps

1. web -> mini-game [call/http]: POST /mini-game/sessions: mời bạn chơi
2. mini-game -> friend [call/di-call]: Hỏi areFriends(userId, friendUserId) (Nếu không phải bạn thì ném DomainException NOT_FRIEND (404) trước khi chạm DB.) {apps/core-api/src/modules/mini-game/mini-game.service.ts:71}
3. mini-game -> postgres [call/shared-db]: Chuẩn hoá cặp user rồi tìm ván đang chờ của cặp này (Nếu đã có ván chờ thì trả luôn, không tạo mới.) {apps/core-api/src/modules/mini-game/mini-game.service.ts:301}
4. mini-game -> postgres [call/shared-db]: Lưu MiniGameSession và 2 dòng MiniGameActiveParticipant (Unique violation thì rollback cả transaction rồi đọc lại cặp để phân biệt replay với 409 ALREADY_WAITING.) {apps/core-api/src/modules/mini-game/mini-game.service.ts:86}
5. mini-game -> redis [async/redis-pubsub]: Publish MiniGameSessionStarted cho cả hai user {apps/core-api/src/modules/mini-game/mini-game.service.ts:130, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
6. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
7. signaling-gateway -> web [async/socket]: Đẩy minigame.session.started xuống socket của hai bạn {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
8. mini-game -> web [return/http]: Trả phiên vừa tạo cho người mời
9. web -> mini-game [call/http]: POST /mini-game/sessions/:id/moves: nộp move
10. mini-game -> postgres [call/shared-db]: Đọc phiên, kiểm tra người nộp là participant {apps/core-api/src/modules/mini-game/mini-game.service.ts:313}
11. mini-game -> postgres [call/shared-db]: UPDATE move có điều kiện; đủ 2 move thì chốt thắng thua, xoá participant (Update 0 dòng thì 409 MOVE_ALREADY_SUBMITTED. Mới 1 move thì không resolve và không publish.) {apps/core-api/src/modules/mini-game/mini-game.service.ts:155}
12. mini-game -> redis [async/redis-pubsub]: Publish MiniGameSessionResolved (2 move + người thắng) cho cả hai (Chỉ chạy khi ván đã resolved.) {apps/core-api/src/modules/mini-game/mini-game.service.ts:232, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
13. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận envelope kết quả qua realtime:user:* {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
14. signaling-gateway -> web [async/socket]: Đẩy kết quả thắng/thua xuống socket của hai bạn {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
15. mini-game -> web [return/http]: Trả phiên đã cập nhật cho người vừa nộp

## Invariants

- Phiên và 2 dòng MiniGameActiveParticipant được ghi trong cùng 1 transaction, nên unique violation rollback cả hai và không để lại row mồ côi; mỗi user chỉ chờ move ở 1 ván.
- Nộp move dùng UPDATE có điều kiện (move IS NULL và status waiting_moves) trong transaction, nên double-submit hoặc ván đã chốt trả 409 và không đổi được move đã nộp.
- Chỉ publish realtime sau khi transaction đã xong; khi mới có 1 move thì không publish gì, nên không lộ move của đối phương.
- Friend sở hữu quan hệ bạn bè, nên mini-game phải hỏi friend.areFriends trước khi tạo phiên thay vì tự đọc bảng bạn bè.

## Failure modes

- when Hai người chưa là bạn hoặc tự mời chính mình: Ném DomainException NOT_FRIEND (404), không ghi DB, không publish.
- when Request khác vừa tạo phiên trước (unique violation): Transaction rollback; đọc lại cặp, nếu đúng cặp thì trả phiên cũ, nếu không thì 409 ALREADY_WAITING.
- when Nộp move lần 2 hoặc ván không còn chờ move: UPDATE 0 dòng, ném 409 MOVE_ALREADY_SUBMITTED, transaction rollback.
- when Redis publish thất bại: publishRealtimeEvent bắt lỗi và chỉ logger.warn; DB đã commit và HTTP vẫn trả thành công, client dùng polling fallback.

## Endpoints covered

- POST /mini-game/sessions [module:mini-game] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mini-game.md
- POST /mini-game/sessions/:id/moves [module:mini-game] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mini-game.md
