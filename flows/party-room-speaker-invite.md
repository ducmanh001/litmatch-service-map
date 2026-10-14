# Mời lên nói và phản hồi lời mời

id: flow:party-room-speaker-invite
group: Phòng party & an toàn
services: module:party-room, module:signaling-gateway, module:media-server

Host mời một khán giả lên speaker; khán giả chấp nhận hoặc từ chối. Chấp nhận phải đổi role trong PostgreSQL và cấp quyền publish thật trên LiveKit SFU, còn thông báo tới client đi qua Redis và signaling-gateway nên nhiều thành phần phải phối hợp.

Lời mời được lưu trong DB rồi báo realtime; chấp nhận đổi role, cấp quyền publish trên SFU và phát role mới cho cả phòng.

## Participants

- web (client): Web client
- party-room (service): Party room
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway
- media-server (service): LiveKit SFU

## Steps

1. web -> party-room [call/http]: Host gọi POST /party/rooms/:id/members/:userId/speaker-invite
2. party-room -> postgres [call/shared-db]: Transaction: khóa phòng, kiểm tra host và target là audience, đặt speakerInvitePending {apps/core-api/src/modules/party-room/party-room.service.ts:514}
3. party-room -> redis [async/redis-pubsub]: Publish PartySpeakerInviteReceived cho target (chỉ khi lời mời mới) (publishRealtimeEvent bắt lỗi publish và chỉ ghi log cảnh báo, không làm hỏng request.) {apps/core-api/src/modules/party-room/party-room.service.ts:559}
4. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
5. signaling-gateway -> web [async/socket]: Emit lời mời tới socket room của target {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
6. party-room -> web [return/http]: Trả thành viên đã có speakerInvitePending
7. web -> party-room [call/http]: Target gọi POST /party/rooms/:id/speaker-invite/accept
8. party-room -> postgres [call/shared-db]: Transaction: kiểm tra lời mời, đếm speaker so với speakerLimit, đổi role thành Speaker {apps/core-api/src/modules/party-room/party-room.service.ts:572}
9. party-room -> media-server [call/sdk]: Sau commit: updateParticipantPublish canPublish=true, chờ ACK từ SFU (Nếu SFU lỗi, role được hoàn lại trong transaction mới và request trả 503.) {apps/core-api/src/modules/party-room/party-room.service.ts:1083}
10. media-server -> party-room [return/sdk]: ACK cấp quyền publish (updated hoặc not_connected)
11. party-room -> postgres [call/shared-db]: Đọc danh sách thành viên còn trong phòng (Nằm ngoài try/catch của publish: lỗi đọc roster không bị nuốt, accept trả lỗi dù role đã đổi và SFU đã cấp quyền.) {apps/core-api/src/modules/party-room/party-room.service.ts:1321}
12. party-room -> redis [async/redis-pubsub]: Publish PartyRoleChanged cho từng thành viên trong phòng {apps/core-api/src/modules/party-room/party-room.service.ts:1326}
13. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận envelope role mới qua PSUBSCRIBE {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
14. signaling-gateway -> web [async/socket]: Emit role mới tới các thành viên trong phòng {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
15. party-room -> web [return/http]: Trả thành viên đã là Speaker
16. web -> party-room [call/http]: Hoặc target gọi POST /party/rooms/:id/speaker-invite/decline
17. party-room -> postgres [call/shared-db]: Transaction: xóa speakerInvitePending, không đổi role, không gọi SFU {apps/core-api/src/modules/party-room/party-room.service.ts:640}
18. party-room -> web [return/http]: Trả thành viên (idempotent)

## Invariants

- Lời mời chỉ đặt speakerInvitePending, role vẫn là audience và token vẫn canPublish=false; chỉ target gọi accept mới đổi role, nên client không tự bypass được consent.
- Việc đếm speaker và đổi role chạy trong cùng transaction có khóa phòng, nên giới hạn speakerLimit không bị vượt khi nhiều lời mời được chấp nhận cùng lúc.
- Quyền publish được cập nhật trên SFU sau khi commit DB và chờ ACK, vì enforce thật nằm ở LiveKit; DB và SFU không được lệch nhau.

## Failure modes

- when LiveKit updateParticipantPublish lỗi khi accept: Role được hoàn về previousRole nếu không có chuyển role mới hơn, SFU được yêu cầu khôi phục grant (lỗi chỉ ghi log) và API trả 503 PARTY_MEDIA_PERMISSION_UPDATE_FAILED.
- when Redis publish lỗi: publishRealtimeEvent chỉ ghi cảnh báo, request vẫn thành công; client có thể không nhận ngay sự kiện realtime.
- when Speaker đã đủ speakerLimit khi accept: Transaction ném SPEAKER_LIMIT_REACHED (409), role không đổi và không gọi SFU.
- when Caller không phải host hoặc target không phải audience khi mời: Transaction ném lỗi NOT_HOST (403) hoặc SPEAKER_INVITE_TARGET_INVALID (409), không có thay đổi nào.
- when Target accept khi không còn lời mời (hoặc không phải audience): Transaction ném SPEAKER_INVITE_NOT_PENDING (409); nếu đã là Speaker thì trả luôn, không gọi SFU và không publish.

## Endpoints covered

- POST /party/rooms/:id/members/:userId/speaker-invite [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/rooms/:id/speaker-invite/accept [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/rooms/:id/speaker-invite/decline [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
