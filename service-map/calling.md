# Service map: Calling

id: service-map:calling
module: module:calling
deployable: core-api
confidence: high

Vòng đời CallSession voice 2 người trên LiveKit

Sở hữu CallSession (pending/active/ended), VoiceMatchReaction, token join LiveKit và bộ đếm thời gian Voice Match enforce ở server (7 phút, pending timeout, reconnect grace). KHÔNG sở hữu MatchSession (Matching ghi), Friendship (Friend ghi), cũng không chạy SFU (media-server) và không có export cho module khác gọi vào.

Owns: bảng call_sessions (status, roomName, joinedA/BAt, startedAt, reconnectStartedAt, endReason, durationSeconds, callKind); bảng voice_match_reactions (unique callId + raterUserId); room LiveKit tên call-{callSessionId} (tạo ngầm khi participant đầu tiên vào, xoá khi end)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/calling/match-sessions/:matchSessionId/join | jwt | Member của voice MatchSession đã confirmed tạo/lấy CallSession (unique match_session_id) và nhận token LiveKit TTL ngắn + livekitUrl theo region; call đã ended trả 409 CALL_ENDED. Throttle 20/phút. |
| e2 | rest | POST | /api/v1/calling/friends/:friendUserId/join | jwt | Mở/lấy friend call lâu dài (canonical pair, không MatchSession, không free timer) chỉ khi hai bên follow nhau và không block, rồi mint token. Throttle 20/phút; không đủ điều kiện trả 404 chung. |
| e3 | rest | GET | /api/v1/calling/calls/:id | jwt | Poll fallback của call.ended: trả trạng thái call + liked/matched/friendUserId, chỉ member (404 gộp chống oracle). Không thuần đọc: gọi LiveKit listParticipants để đối soát phòng khi webhook mất và có thể chuyển call sang joined/active hoặc ended ngay trong r… |
| e4 | rest | POST | /api/v1/calling/calls/:id/end | jwt | Member chủ động kết thúc call, idempotent (lock FOR UPDATE), chốt luôn MatchSession voice ở Matching, dọn room và phát call.ended. |
| e5 | rest | POST | /api/v1/calling/match-sessions/:matchSessionId/end | jwt | Rời màn Voice Match: end call nếu đã có, rồi chốt MatchSession ended kể cả khi CallSession chưa được tạo. |
| e6 | rest | POST | /api/v1/calling/calls/:id/like | jwt | Ghi consent like immutable khi call active/ended (khoá CallSession, unique reaction); đủ like hai chiều thì tạo Friendship + Conversation trong CÙNG transaction. Friend call bị chặn. |
| e7 | webhook | POST | /api/v1/calling/webhooks/livekit | webhook-signature | Nhận webhook LiveKit (participant_joined/left/connection_aborted, room_finished): verify chữ ký JWT trên raw body rồi mới cập nhật state; room không phải call-* bị bỏ qua, luôn 200 khi đã verify. |
| e8 | socket-emit | EMIT | call.ended | internal | Realtime call.ended {callId, matchSessionId, reason, durationSeconds} gửi cho cả hai user sau khi call chuyển ended (best-effort, poll GET /calling/calls/:id là fallback). |
| e9 | job | WORKER | CallTickerService.onApplicationBootstrap | internal | Ticker chạy mỗi CALLING_TICKER_INTERVAL_MS (một instance một lúc nhờ advisory lock): end pending_timeout, end sau reconnect grace, end free_limit khi Voice Match vượt CALLING_FREE_CALL_SECONDS. |

## Coordination edges (why this service talks to others)

### gate follow + tạo Friendship (c1) with friend [service, out, di-call]

- endpoints: e2, e3, e4, e6
- what: ProfileSocialService.canCall(userId, friendUserId) cho friend call; FriendService.ensureFriendship(manager, a, b, VoiceMatch) khi mutual like; FriendService.areFriends để tính matched/friendUserId.
- why: Quan hệ follow/block/bạn bè do Friend sở hữu nên Calling không được tự tính; ensureFriendship nhận EntityManager của transaction like để Friendship + Conversation được tạo cùng lúc với VoiceMatchReaction, không bao giờ có Friendship thiếu Conversation.
- if missing: Người bị block hoặc không follow nhau vẫn mint được token gọi; mutual like có thể để lại reaction mà không có Friendship, hoặc Friendship không có Conversation khi retry.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:208, apps/core-api/src/modules/calling/calling.service.ts:297, apps/core-api/src/modules/calling/calling.service.ts:404
- review: sound — ensureFriendship insert Friendship + Conversation orIgnore trên cùng manager; canCall kiểm block 2 chiều rồi mutual follow.

### đọc/chốt MatchSession voice (c2) with matching [service, out, di-call]

- endpoints: e1, e3, e4, e5, e6, e9
- what: MatchingService.findSessionById (kiểm member/matchType=voice/status=confirmed), endVoiceSession (user rời) và endVoiceSessionForCall(sessionId, userA, userB) sau khi call terminal.
- why: Matching là module DUY NHẤT được ghi MatchSession; call kết thúc từ webhook/ticker/nút end đều phải chốt session ended TRƯỚC khi phát call.ended để user quay lại tìm match không bị session confirmed cũ giữ lại (calling.service.ts:685-692). Hệ quả thứ tự: nếu bước chốt này ném lỗi thì deleteRoom và call.ended bị bỏ qua và không retry (xem notes). Join cũng re-check session sau khi tạo CallSession để chặn race với endMatchSession.
- if missing: User kẹt trong Voice Match session cũ không vào được queue mới; call pending orphan nằm chờ ticker timeout khi đối phương đã rời; join được cho session không còn hợp lệ.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:105, apps/core-api/src/modules/calling/calling.service.ts:167, apps/core-api/src/modules/calling/calling.service.ts:317
- review: sound — cleanupEndedCall await endVoiceSessionForCall không bắt lỗi trước deleteRoom/publish; call đã commit Ended nên không retry — đúng.

### đọc User.region (c3) with user [service, out, di-call]

- endpoints: e1, e2
- what: UserService.getByIdOrThrow(userAId/userBId) để lấy region rồi resolveLivekitUrl(LIVEKIT_REGION_URLS, LIVEKIT_URL, userA.region).
- why: Region thuộc User; để cả hai client của cùng một call nối cùng endpoint LiveKit, URL phải tất định theo region của call.userAId (ADR 0005). Chỉ tra khi bật multi-region, lỗi lookup luôn fallback LIVEKIT_URL nên không chặn join.
- if missing: Hai bên có thể nhận hai URL khác nhau hoặc luôn dùng URL mặc định, tăng latency khi triển khai nhiều region.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:583, apps/core-api/src/modules/calling/calling.service.ts:584
- review: sound — Chỉ tra user khi hasLivekitRegionUrls, dùng region userA, catch rồi fallback LIVEKIT_URL — khớp code.

### mint token, xoá room (c4) with media-server [service, out, sdk]

- endpoints: e1, e2, e3, e4, e5, e6, e9
- what: LivekitRoomPort -> LivekitSdkClient: mintJoinToken(roomName, userId, ttl) khi join, deleteRoom khi end, listParticipantIdentities khi GET call, receiveWebhook để verify chữ ký.
- why: Âm thanh đi thẳng client <-> LiveKit nên chỉ core-api (giữ API secret) mới được cấp quyền vào room: identity = userId từ JWT, room do server đặt tên nên client không tự chọn. Phải xoá room khi end để không leak tài nguyên SFU và đối soát listParticipants vì webhook có thể mất.
- if missing: Client không có token nên không vào được phòng; room SFU không được dọn; webhook mất thì call kẹt pending tới CALLING_PENDING_TIMEOUT_SECONDS rồi bị ticker end dù hai bên đang nói (chỉ GET có reconcile listParticipants mới cứu được).
- evidence: apps/core-api/src/modules/calling/calling.service.ts:579, apps/core-api/src/modules/calling/calling.service.ts:591, apps/core-api/src/modules/calling/clients/livekit-room.adapter.ts:21
- review: corrected — ifMissing 'treo vô hạn' sai: ticker vẫn end pending_timeout / free_limit / reconnect-grace.

### phát call.ended (c5) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e8
- what: publishRealtimeEvent(redis, logger, userId, envelope) với RealtimeEvents.CallEnded {callId, matchSessionId, reason, durationSeconds} cho userA và userB.
- why: Khi ticker/webhook/đối phương kết thúc, client của bên còn lại không có request nào đang chờ nên cần được đẩy sự kiện để dừng màn hình call ngay; core-api chỉ publish vào realtime:user:{userId}, signaling-gateway giữ socket và relay. Publish chạy SAU commit của endById và SAU khi Matching đã chốt session; best-effort (lỗi publish bị nuốt, không rollback) vì GET /calling/calls/:id polling là fallback. Nếu bước chốt MatchSession ném lỗi thì event không bao giờ được publish (xem notes).
- if missing: Bên còn lại chỉ biết call đã ended sau vòng poll kế tiếp, tức UI treo/đồng hồ chạy tiếp vài giây.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:710
- review: sound — publish sau endById commit và sau endVoiceSessionForCall; publishRealtimeEvent catch lỗi chỉ log warn; lỗi chốt session làm bỏ qua event.

### call_sessions + reactions (c7) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e9
- what: Đọc/ghi CallSession (status, mốc joined/started/reconnect, endReason, durationSeconds) và VoiceMatchReaction; transaction + pessimistic_write lock trên call row; ticker quét theo status/created_at.
- why: State machine phải bền và serialize được: lock FOR UPDATE là chốt chặn race end-vs-tick và webhook retry/out-of-order, unique match_session_id đảm bảo 1 session = 1 call, unique (callId, raterUserId) làm like idempotent. durationSeconds lấy theo giờ server, không tin client hay LiveKit.
- if missing: Hai webhook/ticker/nút end cùng lúc có thể end đôi (cleanup + call.ended bắn hai lần), tạo hai call cho một session hoặc đếm sai thời lượng 7 phút.
- evidence: apps/core-api/src/modules/calling/entities/call-session.entity.ts:35, apps/core-api/src/modules/calling/entities/voice-match-reaction.entity.ts:10
- review: sound — pessimistic_write trong endById/markJoined/likeCall, unique match_session và (callId, raterUserId), durationSeconds theo giờ server đều đã xác nhận.

### webhook participant/room (c50) with media-server [service, in, webhook]

- endpoints: e7
- what: LiveKit POST http://host.docker.internal:3000/api/v1/calling/webhooks/livekit (livekit.yaml) với event participant_joined, participant_left, participant_connection_aborted, room_finished kèm Authorization JWT ký bằng API key. Cùng sự kiện còn tới /party/webhooks/livekit vì livekit.yaml khai báo cả hai url; calling bỏ qua room không phải call-*.
- why: Chỉ SFU biết người dùng thực sự đã vào/ra phòng; CallSession chỉ chuyển pending -> active (startedAt, mốc đếm 7 phút) khi cả hai identity joined, và mở reconnect grace khi rớt. Webhook phải nằm ở core-api vì state + timer thuộc core-api, SFU không có business logic.
- if missing: Call kẹt pending tới pending_timeout dù hai bên đang nói, đồng hồ 7 phút không bắt đầu, rớt kết nối không được phát hiện (chỉ còn backstop listParticipants khi có GET).
- evidence: apps/media-server/livekit.yaml:27, apps/core-api/src/modules/calling/webhooks/livekit-webhook.controller.ts:35
- review: corrected — livekit.yaml dùng http://host.docker.internal:3000, không phải core-api:3000.

### chung API key LiveKit (c51) with media-server [service, both, config]

- endpoints: e1, e7
- what: LIVEKIT_API_KEY/LIVEKIT_API_SECRET của core-api phải trùng LIVEKIT_KEYS của media-server; webhook của LiveKit ký bằng cùng key (LIVEKIT_WEBHOOK_API_KEY).
- why: Token join do core-api ký và webhook do LiveKit ký đều là JWT dùng chung một cặp key; core-api verify webhook bằng chính key này nên hai deployable bắt buộc cấu hình đồng bộ.
- if missing: Lệch key: LiveKit từ chối token join, hoặc core-api trả 401 cho mọi webhook nên call không bao giờ active.
- evidence: apps/media-server/docker-compose.yml:13
- review: sound — docker-compose: LIVEKIT_KEYS và LIVEKIT_WEBHOOK_API_KEY lấy từ LIVEKIT_API_KEY; core-api đọc LIVEKIT_API_KEY/SECRET.

### nối thẳng LiveKit bằng token (c52) with web [client, both, sdk]

- endpoints: e1, e2
- what: Web nhận {token, livekitUrl} từ REST join rồi connectMediaRoom(joined.token, joined.livekitUrl) bằng livekit-client để publish mic và phát audio của bên kia.
- why: Media không đi qua core-api; client phải tự mở kết nối WebRTC tới SFU bằng token ngắn hạn core-api cấp, và gọi lại join (token mới) khi rớt mạng vì call chưa ended.
- if missing: Không có luồng âm thanh: core-api chỉ giữ state, không relay audio.
- evidence: apps/web/src/features/voice-match/hooks/use-call-room.ts:94
- review: sound — connectMediaRoom(joined.token, joined.livekitUrl) xác nhận ở use-call-room.ts; core-api không relay audio.

### poll trạng thái call (c53) with web [client, in, http]

- endpoints: e3
- what: useCall poll GET /calling/calls/:id (refetchInterval khi call còn active) để nhận status/endReason khi realtime call.ended không tới.
- why: Realtime best-effort nên REST polling là đường chắc chắn; chính GET này còn kích hoạt đối soát LiveKit nên client phải poll để call pending được chuyển active khi webhook mất.
- if missing: Mất socket hoặc webhook thì UI không biết call đã kết thúc hoặc đã active.
- evidence: apps/web/src/features/voice-match/api.ts:74
- review: sound — GET /calling/calls/:id gọi reconcileCallWithLivekit nên poll thật sự kích hoạt đối soát; useCall có refetchInterval.

### call.ended dừng màn gọi (c54) with web [client, out, socket]

- endpoints: e8
- what: Web nghe RealtimeEvents.CallEnded ở cả màn Voice Match (voice-call-room) lẫn màn friend call (friend-call-room); khi data.callId trùng call đang mở thì đóng phòng LiveKit và chuyển màn kết thúc.
- why: Bên còn lại không có request nào đang chờ khi đối phương/ticker/webhook kết thúc call, nên event call.ended là cách duy nhất để UI dừng ngay; payload chỉ có callId/matchSessionId/reason/durationSeconds nên client so callId rồi mới hành động, và vẫn có poll GET /calling/calls/:id làm fallback.
- if missing: Bên còn lại vẫn thấy đồng hồ và phòng audio chạy cho tới vòng poll kế tiếp, friend call (không có timer tự end) có thể treo UI lâu hơn.
- evidence: apps/web/src/features/voice-match/components/voice-call-room.tsx:200, apps/web/src/features/friend-chat/components/friend-call-room.tsx:94
- review: sound — Cả hai màn so data.callId === callId rồi disconnect và invalidate query — đúng.

## Notes

- Doc là docs/services/calling-service.md (README map tên này; không có calling.md). Doc § 4 và comment trong call-ticker.service.ts vẫn nói 'hai người đã mutual like có thể gọi lại lâu dài', nhưng code (commit ca0067f) và chính § 6 của doc yêu cầu ProfileSocialService.canCall = không block + follow hai chiều; mutual like KHÔNG còn đủ để gọi friend call.
- Doc § 2 mô tả upsert 'ON CONFLICT (match_session_id)'; code thực tế là callRepo.save rồi bắt isUniqueViolation và đọc lại row của bên thắng race (cùng hiệu ứng, khác cách làm).
- Calling KHÔNG tự tạo room LiveKit: LivekitRoomPort của calling chỉ có mintJoinToken/deleteRoom/listParticipantIdentities/receiveWebhook; LivekitSdkClient.createRoom (common/livekit/livekit-sdk.client.ts:74) tồn tại nhưng chỉ party-room dùng (spec gọi với "party-1"). media-server/livekit.yaml không có khối room: nào (không tắt auto_create), nên room được tạo khi participant đầu tiên connect bằng token theo mặc định của LiveKit server (suy luận từ mặc định của LiveKit, không có dòng code hay doc của repo nói thẳng). Calling chỉ xoá room ở cleanupEndedCall.
- Không có thông báo 'đang gọi/ringing' hay web push cho bên kia trong module calling: commit ca0067f (web push) chỉ đổi điều kiện joinFriendCall sang canCall và thêm trang friend-call phía web; người nhận chỉ biết cuộc gọi qua UI/chat, và call.ended là realtime event duy nhất của module.
- Friend call không đi qua free timer nhưng vẫn bị ticker sweepPending (pending quá CALLING_PENDING_TIMEOUT_SECONDS) và sweepReconnectTimeouts end; chỉ nhánh free_limit bị giới hạn call_kind = 'voice_match'.
- CALLING_REDIS chỉ được dùng để publish call.ended (calling-redis.provider.ts) và đóng khi shutdown; không có Redis key/queue/lock riêng của calling. Ticker dùng pg_try_advisory_xact_lock (qua ManagedInterval clusterSingleton) trên Postgres, không phải Redis.
- cleanupEndedCall (calling.service.ts:684-713) chạy tuần tự endVoiceSessionForCall -> deleteRoom (lỗi bị nuốt) -> publish call.ended, và chỉ được gọi từ nhánh justEnded (:477-480), tức đúng 1 lần/call. endVoiceSessionForCall KHÔNG được bọc try/catch và có thể ném lỗi (lỗi DB, hoặc "Call members không khớp Voice Match session ... dữ liệu hỏng" ở matching.service.ts:604). Khi ném, CallSession đã ended trong DB nhưng room SFU không bị xoá, call.ended không được publish và không có lần thử lại (lần end sau thấy call đã ended nên justEnded=false); client chỉ còn poll GET /calling/calls/:id và LiveKit tự dọn room rỗng. Comment "cả 2 đều best-effort" chỉ đúng cho deleteRoom và publish, không đúng cho bước chốt MatchSession.
- media-server/livekit.yaml đăng ký CẢ HAI webhook url (/api/v1/calling/webhooks/livekit và /api/v1/party/webhooks/livekit): mọi sự kiện của mọi room đều được POST tới cả hai controller, mỗi module tự lọc theo prefix room (call-* / party-*) sau khi verify chữ ký. Lệch prefix API_GLOBAL_PREFIX làm LiveKit POST 404 âm thầm (comment trong livekit.yaml, UX audit 2026-07-14).
