# Xem phòng party và bình luận trong phòng

id: flow:party-room-detail-and-comments
group: Phòng party & an toàn
services: module:party-room, module:user, module:signaling-gateway

Client xem danh sách phòng, chi tiết phòng (kèm hồ sơ member lấy từ module user), đọc comment rồi gửi comment với Idempotency-Key. Comment được ghi trong transaction của party-room, sau đó được fanout realtime qua Redis và signaling-gateway tới từng member.

Đọc phòng/comment chỉ đi qua party-room và PostgreSQL (chi tiết phòng gọi thêm user); gửi comment ghi DB trong transaction rồi publish realtime qua Redis tới gateway.

## Participants

- web (client): Web client
- party-room (service): Party Room
- user (service): User
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> party-room [call/http]: GET /party/rooms — danh sách phòng đang mở (cursor) (Mọi route REST đều qua JwtAuthGuard toàn cục trừ route @Public.)
2. party-room -> postgres [call/shared-db]: Query party_rooms rồi đếm member từng phòng {apps/core-api/src/modules/party-room/party-room.service.ts:727, apps/core-api/src/modules/party-room/party-room.service.ts:948}
3. party-room -> web [return/http]: Trả danh sách phòng kèm số member
4. web -> party-room [call/http]: GET /party/rooms/:id — chi tiết phòng
5. party-room -> postgres [call/shared-db]: Đọc phòng và danh sách PartyRoomMember {apps/core-api/src/modules/party-room/party-room.service.ts:685}
6. party-room -> user [call/di-call]: UserService.findByIds lấy nickname của các member {apps/core-api/src/modules/party-room/party-room.service.ts:698}
7. party-room -> web [return/http]: Trả chi tiết phòng + member active
8. web -> party-room [call/http]: GET /party/rooms/:id/comments — comment gần nhất
9. party-room -> postgres [call/shared-db]: Kiểm tra phòng còn Active và user là member đang kết nối (Phòng không tồn tại trả 404, phòng đã đóng trả 409, không phải member trả 403.) {apps/core-api/src/modules/party-room/party-room.service.ts:1245}
10. party-room -> web [return/http]: Trả trang comment (cursor theo seq)
11. web -> party-room [call/http]: POST /party/rooms/:id/comments kèm Idempotency-Key (Throttle 60 request/phút; nội dung được trim và kiểm tra độ dài trước khi mở transaction.)
12. party-room -> postgres [call/shared-db]: Transaction: khoá phòng, kiểm tra member, insert comment (Insert append-only; unique idempotency ở DB. Trùng key cùng nội dung thì trả lại comment cũ và không bắn lại event.) {apps/core-api/src/modules/party-room/party-room.service.ts:799, apps/core-api/src/modules/party-room/party-room.service.ts:801}
13. party-room -> postgres [call/shared-db]: Sau commit: đọc member chưa rời phòng để fanout (Nằm ngoài try/catch của publish: lỗi đọc member không bị nuốt, request lỗi dù comment đã commit.) {apps/core-api/src/modules/party-room/party-room.service.ts:1321}
14. party-room -> redis [async/redis-pubsub]: Publish PartyCommentCreated tới kênh realtime của từng member (publishRealtimeEvent bọc try/catch, lỗi chỉ log warn.) {apps/core-api/src/modules/party-room/party-room.service.ts:1326, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
15. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
16. signaling-gateway -> web [async/socket]: Emit party comment tới socket room của member {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
17. party-room -> web [return/http]: Trả comment vừa tạo (hoặc comment cũ khi replay)

## Invariants

- Membership được kiểm tra dưới lock phòng trong cùng transaction với insert comment, để REST leave và gửi comment đồng thời có thứ tự xác định.
- Event realtime chỉ được publish sau khi transaction commit; replay cùng Idempotency-Key không đi qua bước publish nên không bắn lại event.
- Gateway chỉ relay envelope từ kênh per-user realtime:user:*; party-room phải publish riêng cho từng member chưa rời phòng.

## Failure modes

- when Redis publish thất bại khi fanout comment: Lỗi bị bắt và chỉ log warn; comment đã commit và request vẫn trả thành công, client dựa vào polling GET comments.
- when Cùng Idempotency-Key nhưng khác phòng, người gửi hoặc nội dung: Vi phạm unique idempotency, trả 409 COMMENT_IDEMPOTENCY_CONFLICT.
- when User không phải member đang kết nối khi gửi hoặc đọc comment: Trả 403 NOT_A_MEMBER; gửi comment thì transaction bị rollback.
- when Nội dung comment rỗng hoặc vượt PARTY_COMMENT_MAX_LENGTH: Trả 422 COMMENT_INVALID trước khi mở transaction.
- when Đọc danh sách member để fanout lỗi sau khi comment đã commit: Lỗi không bị nuốt (chỉ publishRealtimeEvent mới bọc try/catch) nên request trả lỗi dù comment đã được lưu; retry cùng Idempotency-Key trả lại comment cũ.

## Endpoints covered

- GET /party/rooms [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- GET /party/rooms/:id [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- GET /party/rooms/:id/comments [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/rooms/:id/comments [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
