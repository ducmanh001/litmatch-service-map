# Vòng đời phòng party với LiveKit

id: flow:party-room-lifecycle
group: Phòng party & an toàn
services: module:party-room, module:user, module:media-server, module:signaling-gateway

Phòng party có DB (party-room) là nguồn sự thật về thành viên, còn âm thanh chạy qua LiveKit SFU (media-server). Hai bên chỉ biết nhau qua token, lệnh SDK và webhook, nên core-api phải giữ DB và SFU khớp nhau, rồi báo realtime qua signaling-gateway.

Tạo phòng, vào phòng bằng token LiveKit, webhook báo host rớt, rồi job dọn phòng đóng phòng và xoá room trên SFU.

## Participants

- web (client): Web client
- party-room (service): party-room (core-api)
- user (service): user
- media-server (service): LiveKit SFU
- signaling-gateway (service): signaling-gateway
- postgres (infra): PostgreSQL
- redis (infra): Redis

## Steps

1. web -> party-room [call/http]: POST /party/rooms: host tạo phòng (Mọi route REST đều qua JwtAuthGuard toàn cục trừ route @Public.) {apps/core-api/src/app/app.module.ts:154}
2. party-room -> user [call/di-call]: Chỉ khi bật multi-region: lấy hồ sơ host để chọn LiveKit URL (resolveHostLivekitUrl chạy trước transaction; nếu chưa cấu hình LIVEKIT_REGION_URLS thì trả LIVEKIT_URL mặc định mà không gọi User. URL được snapshot vào row phòng.) {apps/core-api/src/modules/party-room/party-room.service.ts:1155}
3. party-room -> postgres [call/shared-db]: Lưu phòng và membership của host trong transaction {apps/core-api/src/modules/party-room/party-room.service.ts:162}
4. party-room -> media-server [call/sdk]: Tạo room tường minh trên SFU trước khi ai join (createRoom chạy sau transaction lưu phòng. Nếu createRoom throw, PartyRoomService.createRoom gọi closeRoomById(room.id, Error) để bù (đóng phòng vừa tạo) rồi trả 503 MEDIA_ROOM_CREATE_FAILED; đây là nhánh lỗi, không phải đường chạy thường.) {apps/core-api/src/modules/party-room/party-room.service.ts:216}
5. web -> party-room [call/http]: POST /party/rooms/:id/join: vào phòng
6. party-room -> postgres [call/shared-db]: Khoá phòng đang hoạt động, ghi membership trong transaction {apps/core-api/src/modules/party-room/party-room.service.ts:243}
7. party-room -> redis [async/redis-pubsub]: Publish PartyMemberJoined tới thành viên còn lại (trừ người vừa vào) (Chỉ publish PartyMemberJoined khi isNewJoin. Join lại thì publish PartyHostReconnected (host quay lại) hoặc PartyMemberReconnected (thành viên quay lại) nếu người đó đang bị đánh dấu rớt; publishToRoomMembers đọc danh sách member rồi publish từng user, loại excludeUserId.) {apps/core-api/src/modules/party-room/party-room.service.ts:1326, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11, apps/core-api/src/modules/party-room/party-room.service.ts:340}
8. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
9. signaling-gateway -> web [async/socket]: Relay envelope xuống socket của từng user {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
10. party-room -> web [return/http]: Trả livekitUrl và token join do server cấp (Token do PartyLivekitRoomPort.mintJoinToken ký, identity do server đặt, canPublish theo role trong DB.) {apps/core-api/src/modules/party-room/ports/livekit-party-room.ts:30}
11. media-server -> party-room [async/webhook]: Webhook LiveKit báo participant rời, party-room verify chữ ký {apps/core-api/src/modules/party-room/ports/livekit-party-room.ts:57}
12. party-room -> postgres [call/shared-db]: Đánh dấu host rớt và bắt đầu grace trong transaction (Sau commit, chỉ khi vừa đánh dấu mới publish PartyHostDisconnected tới thành viên, cùng đường Redis như bước 7-9. Với thành viên thường, webhook mở transaction riêng khoá phòng và đặt disconnectedAt rồi publish PartyMemberDisconnected; webhook retry không dời mốc và không bắn event đôi.) {apps/core-api/src/modules/party-room/party-room.service.ts:1282, apps/core-api/src/modules/party-room/party-room.service.ts:1295, apps/core-api/src/modules/party-room/party-room.service.ts:1231}
13. party-room -> postgres [call/shared-db]: Job host-grace: quét phòng quá grace, closeRoomById đóng phòng trong transaction (PartyRoomSweeperService.sweepExpiredHostGrace tìm phòng Active có hostDisconnectedAt quá PARTY_HOST_DISCONNECT_GRACE_SECONDS rồi gọi closeRoomById(HostLeft) với guard re-check trên row đã lock; transaction khoá phòng, set Closed, cập nhật leftAt của member.) {apps/core-api/src/modules/party-room/party-room.service.ts:1014, apps/core-api/src/modules/party-room/jobs/party-room-sweeper.service.ts:128}
14. party-room -> media-server [call/sdk]: Sau khi DB đóng: deleteRoom trên SFU (lỗi bị nuốt, chỉ log debug) (PartyRoomSweeperService chạy runHostGraceCheckOnce theo ManagedInterval (PARTY_HOST_GRACE_CHECK_INTERVAL_MS); closeRoomById chỉ gọi deleteRoom khi transaction thực sự chuyển phòng sang Closed.) {apps/core-api/src/modules/party-room/ports/livekit-party-room.ts:51, apps/core-api/src/modules/party-room/jobs/party-room-sweeper.service.ts:75}
15. party-room -> redis [async/redis-pubsub]: Publish sự kiện phòng đã đóng tới thành viên {apps/core-api/src/modules/party-room/party-room.service.ts:1058}

## Invariants

- Phòng được lưu trong transaction rồi mới tạo room trên SFU, và token join do server mint (identity do server đặt), nên DB là nguồn sự thật về ai là thành viên và role.
- closeRoomById đóng phòng trong transaction DB trước, rồi mới xoá room trên SFU và publish realtime, nên thành viên không thấy phòng đã đóng khi DB chưa ghi.
- hostDisconnectedAt chỉ được set bởi webhook participant_left của host (markHostDisconnected), nên job host-grace chỉ đóng được phòng mà webhook đã đánh dấu; phòng không có webhook thì chỉ sweepStaleRooms (PARTY_STALE_ROOM_SECONDS, không còn member hoặc SFU roomExists=false) mới dọn.

## Failure modes

- when Webhook LiveKit không tới hoặc sai chữ ký: Sai chữ ký hoặc thiếu header thì controller trả 401 WEBHOOK_INVALID. Nếu webhook host rớt không tới thì hostDisconnectedAt không được set, job host-grace không thấy phòng; chỉ sweepStaleRooms (phòng quá PARTY_STALE_ROOM_SECONDS không có member hoặc SFU đã đóng), webhook room_finished hoặc host gọi leave mới đóng được.
- when Publish Redis realtime thất bại sau khi DB đã ghi: publishRealtimeEvent bắt lỗi và chỉ logger.warn (không throw); DB vẫn đúng, client không nhận push và đọc lại trạng thái qua REST/polling.
- when Xoá room trên SFU thất bại khi đóng phòng: deleteRoom lỗi bị .catch nuốt và chỉ log debug; DB đã đóng nên phòng không còn hoạt động. Suy luận: room SFU còn sót tự hết nhờ emptyTimeout cấu hình lúc createRoom; sweeper chỉ quét phòng Active nên không dọn lại.
- when Tạo room trên SFU thất bại khi host tạo phòng: Service bù bằng closeRoomById(Error) (đóng phòng trong DB) rồi trả 503 MEDIA_ROOM_CREATE_FAILED để client thử lại.

## Uncertain

- Steps 7-9: the gateway-to-web socket hop cites the same psubscribe fact as the Redis-to-gateway hop.

## Endpoints covered

- POST /party/rooms [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/rooms/:id/join [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- POST /party/webhooks/livekit [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
- PartyRoomSweeperService.onApplicationBootstrap [module:party-room] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/party-room.md
