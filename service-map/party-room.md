# Service map: Party Room

id: service-map:party-room
module: module:party-room
deployable: core-api
confidence: high

Phòng audio nhiều người, role và grace rớt mạng

Sở hữu phòng party (party_rooms), membership theo dòng, comment append-only, lời mời speaker và vòng đời phòng (host rời/rớt, sweeper, grace reconnect). Quyền publish audio được enforce ở LiveKit SFU qua token/updateParticipant; KHÔNG sở hữu hồ sơ user (User), ví/quà (Gift/Economy) và không phải media-server.

Owns: bảng party_rooms (speaker_limit + livekit_url snapshot lúc tạo, host_disconnected_at, close_reason); bảng party_room_members (partial unique: 1 membership active/(room,user) và 1 phòng active/user); bảng party_room_comments (append-only, unique idempotency_key); LiveKit room tên party-{roomId} (tạo/xoá tường minh qua port)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/party/rooms | jwt | Tạo phòng: transaction lưu party_rooms + membership host (speaker_limit và livekit_url snapshot), rồi createRoom LiveKit tường minh và mint token host; SFU fail thì đóng phòng (error) + 503; throttle 5/phút, 1 user chỉ 1 phòng active. |
| e2 | rest | GET | /api/v1/party/rooms | jwt | Liệt kê phòng active kèm số member, phân trang cursor, lọc q/category; chỉ đọc DB. |
| e3 | rest | GET | /api/v1/party/rooms/:id | jwt | Chi tiết phòng + member active, nickname lấy từ User (findByIds); là nguồn sự thật mà client poll fallback để biết hostDisconnectedAt/status. |
| e4 | rest | GET | /api/v1/party/rooms/:id/comments | jwt | Lấy comment mới nhất theo cursor (đi lùi); chỉ member của phòng mới đọc được. |
| e5 | rest | POST | /api/v1/party/rooms/:id/comments | jwt | Gửi comment: bắt buộc Idempotency-Key (unique DB, replay không bắn lại), throttle 60/phút, kiểm member active dưới lock phòng rồi publish party.comment.created sau commit. |
| e6 | rest | POST | /api/v1/party/rooms/:id/join | jwt | Join làm audience (token canPublish=false) hoặc rejoin: đếm PARTY_MAX_MEMBERS dưới lock; rejoin của host/member trong grace xoá cờ disconnected và bắn party.host.reconnected / party.member.reconnected; token mint từ role trong DB. |
| e7 | rest | POST | /api/v1/party/rooms/:id/leave | jwt | Rời phòng idempotent: member thường nhả membership + removeParticipant khỏi SFU; host rời chủ động thì đóng phòng ngay (host_left) và báo party.room.closed. |
| e8 | rest | POST | /api/v1/party/rooms/:id/members/:userId/role | jwt | Host thu quyền speaker về audience dưới lock phòng; role commit trước rồi updateParticipantPublish(canPublish=false) ngoài transaction, lỗi SFU thì rollback role + 503; chỉ host (NOT_HOST 403). |
| e9 | rest | POST | /api/v1/party/rooms/:id/members/:userId/speaker-invite | jwt | Host mời audience lên speaker: chỉ ghi speaker_invite_pending và bắn party.speaker.invite.received tới target; chưa đổi role hay quyền publish. |
| e10 | rest | POST | /api/v1/party/rooms/:id/speaker-invite/accept | jwt | Target chấp nhận lời mời: đếm speaker so speaker_limit dưới lock phòng, đổi role, updateParticipantPublish(true), bắn party.role.changed; chỉ chính target mới đổi được role của mình. |
| e11 | rest | POST | /api/v1/party/rooms/:id/speaker-invite/decline | jwt | Target từ chối lời mời speaker (idempotent), chỉ xoá cờ speaker_invite_pending. |
| e12 | webhook | POST | /api/v1/party/webhooks/livekit | webhook-signature | Nhận webhook LiveKit (participant_left, room_finished) cho room party-*: verify chữ ký JWT trên raw body, sai thì 401; lọc prefix party-, idempotent, set disconnected_at/host_disconnected_at hoặc đóng phòng. |
| e13 | socket-emit | EMIT | party.comment.created | internal | Báo comment mới tới mọi member active qua Redis realtime:user:{userId}; chỉ bắn khi insert mới, best-effort sau commit. |
| e14 | socket-emit | EMIT | party.host.disconnected | internal | Báo host rớt kết nối ngoài ý muốn (chỉ bắn khi đặt mốc hostDisconnectedAt lần đầu) để client hiện banner chờ; REST poll vẫn là nguồn sự thật. |
| e15 | socket-emit | EMIT | party.host.reconnected | internal | Báo host đã vào lại trong grace (mốc hostDisconnectedAt được xoá) để client ẩn banner. |
| e16 | socket-emit | EMIT | party.member.disconnected | internal | Báo member thường rớt kết nối kèm disconnectedAt; chưa rời roster, chờ grace. |
| e17 | socket-emit | EMIT | party.member.joined | internal | Báo member mới vào phòng (role audience) cho các member khác, loại trừ chính người join. |
| e18 | socket-emit | EMIT | party.member.left | internal | Báo member rời roster (REST leave hoặc hết grace do sweeper chốt). |
| e19 | socket-emit | EMIT | party.member.reconnected | internal | Báo member rớt mạng đã vào lại trong grace, giữ nguyên membership. |
| e20 | socket-emit | EMIT | party.role.changed | internal | Báo role của một member đổi (host thu quyền hoặc target accept speaker) để client đồng bộ quyền mic. |
| e21 | socket-emit | EMIT | party.room.closed | internal | Báo phòng đã đóng kèm reason (host_left, finished, swept, error, admin_closed) tới các member active ngay trước khi đóng; chỉ lời gọi thực hiện transition mới bắn. |
| e22 | socket-emit | EMIT | party.speaker.invite.received | internal | Báo riêng cho target rằng host vừa mời lên speaker (publish tới realtime:user:{targetUserId}). |
| e23 | job | WORKER | PartyRoomSweeperService.onApplicationBootstrap | internal | PartyRoomSweeperService: 2 interval (sweeper chính 30s: đóng phòng stale không còn member hoặc không còn trên SFU qua roomExists; grace-check 5s: đóng phòng host rớt quá grace và chốt member rớt quá hạn); bọc clusterSingleton (advisory lock Postgres). |

## Coordination edges (why this service talks to others)

### hồ sơ/region của user (c1) with user [service, out, di-call]

- endpoints: e1, e3
- what: UserService.getByIdOrThrow(hostUserId) để lấy User.region của host khi tạo phòng, và UserService.findByIds(memberIds) để lấy nickname cho chi tiết phòng.
- why: Party không sở hữu hồ sơ: nickname chỉ có ở User. Region của host phải do server suy ra từ User.region (không nhận từ client) để ghim livekit_url cho cả phòng theo ADR 0005, cùng nguồn với shard matching.
- if missing: Chi tiết phòng không hiển thị được tên member; khi bật LIVEKIT_REGION_URLS, phòng sẽ không chọn được SFU gần host hoặc phải tin region do client gửi.
- evidence: apps/core-api/src/modules/party-room/party-room.service.ts:698, apps/core-api/src/modules/party-room/party-room.service.ts:1155
- review: sound — resolveHostLivekitUrl đọc host.region và ghim vào livekit_url; findByIds dùng ở getRoom(includeProfiles).

### token, tạo phòng, quyền mic (c2) with media-server [service, out, sdk]

- endpoints: e1, e6, e7, e8, e10, e12, e23
- what: Qua PartyLivekitRoomPort: createRoom(party-{id}, maxParticipants, emptyTimeout), mintJoinToken(ttl, canPublish theo role), updateParticipantPublish, removeParticipant, deleteRoom, roomExists.
- why: Quyền nói phải được enforce ở SFU chứ không tin client: audience nhận token canPublish=false nên tự unmute bị LiveKit chặn; cấp/thu speaker phải đẩy xuống SFU bằng updateParticipantPublish. Chỉ SFU biết room còn sống nên sweeper phải hỏi roomExists.
- if missing: Audience tự bật mic được hoặc speaker mới không nói được; DB còn phòng mà SFU đã empty-timeout thì phòng treo; rời DB mà SFU còn nối thì state lệch.
- evidence: apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:21, apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:30, apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:42
- review: sound — mintToken đặt canPublish chỉ cho Host/Speaker; sweeper dùng roomExists; createRoom có maxParticipants/emptyTimeoutSeconds.

### fanout party.* realtime (c3) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e5, e6, e7, e8, e9, e10, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23
- what: publishRealtimeEvent(realtime:user:{userId}) cho từng member active với 10 event party.* (comment, host/member disconnected-reconnected, joined, left, role.changed, speaker.invite.received, room.closed), payload dựng sẵn.
- why: Mọi member cần thấy roster/role/trạng thái host gần như tức thời; gateway không có business logic và không có room channel nên core-api phải liệt kê member active và publish per-user SAU commit. Là best-effort, REST poll là fallback.
- if missing: Client chỉ còn poll REST (5s khi socket mất): banner host mất kết nối, lời mời speaker và comment đến trễ hoặc không có hiệu ứng realtime.
- evidence: apps/core-api/src/modules/party-room/party-room.service.ts:559, apps/core-api/src/modules/party-room/party-room.service.ts:1058, apps/core-api/src/modules/party-room/party-room.service.ts:1326
- review: sound — publishToRoomMembers/publishRealtimeEvent đều chạy sau khi transaction commit; đúng 10 event party.*.

### phòng/member/comment + lock (c5) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e23
- what: Đọc/ghi party_rooms, party_room_members, party_room_comments; SELECT ... FOR UPDATE trên dòng party_rooms để tuần tự hoá join/leave/role/close; sweeper bọc pg advisory xact lock.
- why: Các bất biến nằm ở DB: partial unique index (1 membership active/(room,user), 1 phòng active/user), unique idempotency_key của comment, và lock dòng phòng chặn race 2 yêu cầu speaker vượt speaker_limit hoặc sweeper đóng nhầm phòng vừa hồi phục.
- if missing: User vào 2 phòng cùng lúc, vượt cap speaker, comment trùng khi retry, hoặc phòng bị đóng nhầm khi host vừa reconnect.
- evidence: apps/core-api/src/modules/party-room/entities/party-room-comment.entity.ts:13, apps/core-api/src/modules/party-room/entities/party-room-member.entity.ts:21, apps/core-api/src/modules/party-room/entities/party-room.entity.ts:41
- review: sound — Đã xác nhận lockActiveRoom/closeRoomById dùng pessimistic_write và ManagedInterval dùng pg_try_advisory_xact_lock.

### admin đóng/liệt kê phòng (c6) with admin [service, in, di-call]

- endpoints: 
- what: AdminService.closePartyRoom gọi closeRoomById(roomId, AdminClosed, undefined, hook) với hook ghi audit log trong CÙNG transaction; còn listRoomsWithMemberCounts và countActiveRooms cho dashboard.
- why: Party là chủ của trạng thái phòng và dọn SFU/publish realtime sau commit; admin chỉ được đóng qua cổng này để audit và việc đóng phòng chung một transaction, còn cleanup LiveKit và party.room.closed vẫn do owner chạy.
- if missing: Admin phải sửa thẳng bảng party_rooms: SFU và client không được báo, hoặc đóng phòng mà thiếu audit log.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:214, apps/core-api/src/modules/admin/admin.service.ts:234, apps/core-api/src/modules/admin/admin.service.ts:637
- review: sound — closeRoomById gọi onTransition(manager, room) trong transaction; deleteRoom và publish chạy sau commit.

### danh sách member phòng (c7) with gift [service, in, di-call]

- endpoints: 
- what: GiftService gọi PartyRoomService.getActiveRoomMembers(roomId) để kiểm người tặng/nhận đều là member active trước khi trừ kim cương, và để fanout gift.sent cho cả phòng.
- why: Danh sách member active chỉ nằm trong party_room_members nên Gift phải hỏi Party (không đọc bảng của module khác). Đây là check nghiệp vụ, không phải chốt an toàn tiền: tiền đúng nhờ transaction và idempotency ở economy.
- if missing: Có thể tặng quà cho người đã rời phòng hoặc ở phòng khác, và hiệu ứng quà không có danh sách người nhận để phát.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:184, apps/core-api/src/modules/gift/gift.service.ts:495
- review: sound — gift.service ghi rõ check nghiệp vụ, không phải chốt an toàn tiền; getActiveRoomMembers ném ROOM_CLOSED nếu phòng đã đóng.

### webhook LiveKit về party (c8) with media-server [service, in, webhook]

- endpoints: e12
- what: LiveKit POST participant_left / room_finished tới /api/v1/party/webhooks/livekit; controller verify chữ ký trên raw body bằng receiveWebhook rồi chỉ xử lý room có prefix party-.
- why: Chỉ SFU biết người dùng rớt kết nối thật hay room đã kết thúc; DB không thể suy ra từ REST. Cần webhook để đặt disconnected_at/host_disconnected_at và đóng phòng. LiveKit có 2 URL webhook (calling và party), nên mỗi module lọc theo prefix.
- if missing: Host rớt mạng không có mốc hostDisconnectedAt nên grace-check không đóng phòng; member rớt vẫn nằm roster. Sweeper 30s chỉ đóng phòng rỗng hoặc SFU đã mất room, không bắt được phòng còn member.
- evidence: apps/media-server/livekit.yaml:28
- review: corrected — Sweeper 30s không đóng được phòng còn member mà SFU còn sống; grace-check phụ thuộc hostDisconnectedAt do webhook đặt.

### LIVEKIT_API_KEY dùng chung (c9) with media-server [service, both, config]

- endpoints: e1, e12
- what: core-api (LIVEKIT_URL/API_KEY/API_SECRET) và livekit.yaml dùng chung một cặp khoá: core-api ký token/gọi RoomService, LiveKit ký webhook bằng key inject từ LIVEKIT_WEBHOOK_API_KEY lấy từ cùng LIVEKIT_API_KEY.
- why: Token join và chữ ký webhook chỉ được tin khi hai bên cùng khoá bí mật; lệch khoá thì SFU từ chối token hoặc core-api từ chối webhook. Party và calling cùng dùng 1 cụm LiveKit nên cùng bộ biến.
- if missing: Token mint ra bị SFU từ chối hoặc webhook bị 401, nên người dùng không vào được phòng và state phòng không cập nhật.
- evidence: apps/media-server/livekit.yaml:22
- review: sound — WebhookReceiver(apiKey, apiSecret) và RoomServiceClient dùng cùng cặp khoá; livekit.yaml chú thích khoá webhook cùng nguồn.

### client nối thẳng LiveKit (c10) with web [client, out, sdk]

- endpoints: e1, e6
- what: Sau REST create/join, web gọi connectMediaRoom(joined.token, joined.livekitUrl) bằng livekit-client để nối trực tiếp tới SFU với token và URL snapshot của phòng.
- why: Audio không đi qua core-api; client phải dùng đúng token (đã khoá canPublish theo role) và livekitUrl đã ghim lúc tạo phòng, kể cả khi người join ở region khác host.
- if missing: REST xong nhưng không có luồng audio; hoặc client tự chọn URL SFU sai region.
- evidence: apps/web/src/features/party-room/hooks/use-party-room-media.ts:87
- review: sound — connectMediaRoom(joined.token, joined.livekitUrl) khớp snapshot livekitUrlOf(room).

### poll REST khi realtime hụt (c11) with web [client, out, http]

- endpoints: e3, e4, e5
- what: Web poll GET /party/rooms/:id và /comments với interval thích nghi (thưa khi socket khoẻ, sát 5s khi mất) và gửi Idempotency-Key khi POST comment.
- why: Realtime chỉ best-effort sau commit nên REST phải đủ để client tự đồng bộ roster, hostDisconnectedAt và comment; Idempotency-Key làm retry comment không nhân đôi.
- if missing: Event realtime rớt làm client thấy roster/comment cũ vô thời hạn; retry mạng sinh comment trùng.
- evidence: apps/web/src/features/party-room/api.ts:167
- review: sound — PARTY_ROOM_DETAIL_REFETCH_INTERVAL_MS=5000 khi mất socket, HEALTHY=30000; mutation gửi header Idempotency-Key.

## Notes

- Doc party-room-service.md §3/§6 nói 'chạy nhiều instance an toàn nhờ closeRoomById idempotent' nhưng code còn bọc 2 interval của PartyRoomSweeperService bằng clusterSingleton (PostgreSQL advisory xact lock) nên mỗi tick chỉ 1 replica làm việc.
- Doc § 1 liệt kê close_reason host_left|finished|swept|error; code còn có admin_closed (PartyRoomCloseReason.AdminClosed) do admin.closePartyRoom gọi closeRoomById kèm audit trong cùng transaction.
- Doc § 8 không liệt kê party.host.disconnected/party.host.reconnected trong danh sách event (chỉ nhắc ở § 4); code emit đủ cả hai.
- Event gift.sent KHÔNG do party-room phát: Gift tự publish qua publishRealtimeEvent sau khi lấy danh sách member từ PartyRoomService.getActiveRoomMembers.
- party-room KHÔNG gọi Safety (block/ban/report) ở bất kỳ endpoint nào (grep Safety/block trong module chỉ ra 0 nơi dùng ngoài spec): user bị block vẫn join cùng phòng và comment được. Đây là khoảng trống so với yêu cầu moderation, cần xác nhận với nghiệp vụ.
- Controller list (GET /party/rooms) và get (GET /party/rooms/:id) chỉ yêu cầu JWT, không kiểm tra là member.
