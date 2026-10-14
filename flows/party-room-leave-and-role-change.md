# Rời phòng và đổi vai trò thành viên

id: flow:party-room-leave-and-role-change
group: Phòng party & an toàn
services: module:party-room, module:media-server, module:signaling-gateway

Member rời phòng party hoặc host thu quyền speaker: party-room đổi trạng thái trong PostgreSQL dưới lock phòng, rồi đẩy quyền publish xuống LiveKit (SFU) và báo roster cho các member qua Redis và signaling-gateway, vì quyền audio phải được enforce ở SFU chứ không chỉ ở DB.

Hình cho thấy leave (member thường hoặc host) và changeRole: DB commit trước, sau đó LiveKit và fanout realtime.

## Participants

- web (client): Web client
- party-room (service): Party room
- postgres (infra): PostgreSQL
- media-server (service): LiveKit SFU
- redis (infra): Redis pub/sub
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> party-room [call/http]: POST /party/rooms/:id/leave (JWT) để rời phòng {apps/core-api/src/modules/party-room/party-room.controller.ts:149}
2. party-room -> postgres [call/shared-db]: Transaction: khóa phòng, đọc membership, member thường thì set leftAt (Lock pessimistic_write cả với member thường để tuần tự với changeRole đang đếm speaker; không còn membership active thì no-op.) {apps/core-api/src/modules/party-room/party-room.service.ts:378}
3. party-room -> postgres [call/shared-db]: Nhánh host rời: closeRoomById đóng phòng, set leftAt mọi member (Chỉ chạy khi membership là Host (left.isHost); sau đó leaveRoom return.) {apps/core-api/src/modules/party-room/party-room.service.ts:1014}
4. party-room -> media-server [async/sdk]: Host rời: deleteRoom trên SFU, lỗi chỉ log debug {apps/core-api/src/modules/party-room/party-room.service.ts:1046}
5. party-room -> redis [async/redis-pubsub]: Publish PartyRoomClosed cho từng member đã active {apps/core-api/src/modules/party-room/party-room.service.ts:1058}
6. party-room -> redis [async/redis-pubsub]: Member thường rời: đọc roster còn lại, publish PartyMemberLeft (publishToRoomMembers đọc memberRepo (leftAt IS NULL) rồi publish từng user; chạy ngay sau commit. Chỉ lệnh publish bị bọc try/catch, lỗi đọc roster không bị nuốt.) {apps/core-api/src/modules/party-room/party-room.service.ts:1326}
7. party-room -> media-server [async/sdk]: Member thường rời: removeParticipant kick khỏi SFU, lỗi bị bỏ qua {apps/core-api/src/modules/party-room/party-room.service.ts:417}
8. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
9. signaling-gateway -> web [async/socket]: Emit sự kiện xuống socket room của từng member {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
10. party-room -> web [return/http]: Trả { ok: true } (leave idempotent)
11. web -> party-room [call/http]: POST /party/rooms/:id/members/:userId/role: host thu quyền speaker {apps/core-api/src/modules/party-room/party-room.controller.ts:163}
12. party-room -> postgres [call/shared-db]: Transaction: khóa phòng, kiểm tra caller là host, đổi role target (Không phải host thì NOT_HOST 403; target là Host thì CONFLICT; role không đổi thì trả luôn, không đụng SFU.) {apps/core-api/src/modules/party-room/party-room.service.ts:432}
13. party-room -> media-server [call/sdk]: Sau commit: updateParticipantPublish canPublish=false, chờ ACK (Gọi LiveKit sau khi transaction đã commit, không giữ lock row trong lúc gọi provider.) {apps/core-api/src/modules/party-room/party-room.service.ts:1083}
14. party-room -> postgres [call/shared-db]: Nhánh lỗi LiveKit: transaction hoàn tác role nếu chưa có transition mới (Chỉ chạy khi updateParticipantPublish throw; sau đó cố khôi phục grant cũ và trả 503 PARTY_MEDIA_PERMISSION_UPDATE_FAILED.) {apps/core-api/src/modules/party-room/party-room.service.ts:1092}
15. party-room -> redis [async/redis-pubsub]: Publish PartyRoleChanged cho mọi member còn trong phòng {apps/core-api/src/modules/party-room/party-room.service.ts:1326}
16. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận PartyRoleChanged {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
17. signaling-gateway -> web [async/socket]: Emit PartyRoleChanged xuống socket room của từng member {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
18. party-room -> web [return/http]: Trả PartyRoomMemberDto với role mới

## Invariants

- Lock row phòng (pessimistic_write) được lấy cả khi member thường rời để tuần tự với changeRole đang đếm speaker, nên số speaker không bị đếm lệch.
- Gọi LiveKit chỉ sau khi transaction commit: lệnh provider không nằm trong tính nguyên tử của PostgreSQL, nên không giữ lock hay để DB rollback làm SFU lệch.
- updateParticipantPublish chờ ACK của SFU rồi mới trả, vì quyền publish phải được enforce ở SFU chứ không tin client.
- Fanout realtime chạy sau commit qua Redis rồi gateway: core-api không nói chuyện trực tiếp với socket, gateway chỉ relay envelope.

## Failure modes

- when LiveKit updateParticipantPublish lỗi khi đổi role: Hoàn tác role trong DB nếu chưa có transition mới, cố khôi phục grant cũ, trả 503 PARTY_MEDIA_PERMISSION_UPDATE_FAILED, không publish PartyRoleChanged.
- when removeParticipant hoặc deleteRoom trên SFU lỗi: Lỗi bị bắt và chỉ log debug; DB đã rời/đóng vẫn giữ nguyên.
- when Publish Redis realtime lỗi: publishRealtimeEvent bắt lỗi và log warn, request vẫn thành công; client còn polling fallback.
- when Người gọi không phải host khi đổi role: Transaction ném NOT_HOST 403, không có thay đổi nào.
- when Transaction hoàn tác role cũng lỗi sau khi LiveKit lỗi: Chỉ log error, không khôi phục grant; vẫn ném 503 PARTY_MEDIA_PERMISSION_UPDATE_FAILED nên DB có thể giữ role Audience trong khi SFU chưa đổi quyền.

## Endpoints covered

- POST /party/rooms/:id/leave [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/rooms/:id/members/:userId/role [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
