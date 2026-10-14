# Service map: Media Server (LiveKit SFU)

id: service-map:media-server
module: module:media-server
deployable: media-server
confidence: high

LiveKit SFU: cấu hình và deploy, không logic

Không phải code NestJS: chỉ là cấu hình/deploy image livekit/livekit-server (livekit.yaml, compose, k8s/base/media-server) làm SFU chuyển tiếp audio/video giữa client; giữ room runtime và trạng thái cụm trong Redis. KHÔNG chứa business logic, KHÔNG truy cập DB: quyền tham gia, membership, timer, billing và giới hạn speaker đều ở core-api (calling / party-room).

Owns: Cấu hình livekit.yaml: cổng 7880 (HTTP/WS + server API), 7881 (TCP RTC), UDP 50000-50200, Prometheus 6789; Room media runtime và participant đang kết nối (trạng thái cụm chia sẻ qua Redis); Danh sách webhook URL trỏ về core-api (calling + party)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e50 | port | TCP | 7880 (HTTP/WS signaling + server API) | other | Cổng điều khiển LiveKit: client WebSocket vào phòng bằng token TTL ngắn; core-api gọi server API (createRoom/deleteRoom/updateParticipant/removeParticipant/listParticipants) trên cùng cổng bằng chữ ký key/secret. Cũng là đích của liveness/readiness probe k8s. |
| e51 | port | TCP | 7881 (RTC over TCP) | other | Cổng RTC TCP dự phòng khi mạng của client chặn UDP; media vẫn đi thẳng client tới SFU, không qua core-api. |
| e52 | port | UDP | 50000-50200 (RTC UDP range) | other | Dải UDP cho media WebRTC của mọi participant trên node; phải mở ra Internet và được client route tới (hostNetwork theo ADR 0005, hoặc use_external_ip/NODE_IP khi chạy dev qua LAN). |
| e53 | port | HTTP | 6789 /metrics (Prometheus) | internal | Cổng metrics nội bộ riêng, tách khỏi control port 7880, để Prometheus scrape và chạy prometheus-alerts.yaml. |
| e54 | port | WEBHOOK-OUT | -> core-api /api/v1/calling/webhooks/livekit | webhook-signature | LiveKit đẩy participant_joined/participant_left/room_finished về CallingService để chuyển call sang active, tính startedAt và xử lý rớt kết nối; ký JWT bằng API key, controller verify trên rawBody trước khi tin. |
| e55 | port | WEBHOOK-OUT | -> core-api /api/v1/party/webhooks/livekit | webhook-signature | LiveKit đẩy participant_left/room_finished về PartyRoomService để phát hiện member/host rớt ngoài ý muốn và đóng phòng; mỗi controller tự lọc theo prefix room (call-* / party-*). |

## Coordination edges (why this service talks to others)

### mint token + xoá phòng gọi (c1) with calling [service, in, sdk]

- endpoints: e50
- what: CallingService mint LiveKit join token TTL CALLING_TOKEN_TTL_SECONDS (mintJoinToken), deleteRoom khi kết thúc cuộc gọi, listParticipantIdentities để kiểm tra ai còn trong phòng, và chọn URL client theo region (resolveLivekitUrl).
- why: SFU chỉ biết room/participant, không biết ai được phép nói chuyện với ai; core-api giữ authz Voice Match nên chỉ nó mới được mint token, và phải xoá room sau khi chốt cuộc gọi để giải phóng tài nguyên SFU.
- if missing: Không ai có token hợp lệ để vào phòng; nếu thiếu deleteRoom thì room treo tới empty-timeout và có thể tiếp tục nghe/nói sau khi call đã kết thúc và billing đã chốt.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:579, apps/core-api/src/modules/calling/calling.service.ts:591
- review: sound — calling.service mint token với CALLING_TOKEN_TTL_SECONDS, deleteRoom ở cleanupEndedCall, listParticipantIdentities ở reconcile, region qua resolveCallLivekitUrl.

### tạo phòng + quyền speaker (c2) with party-room [service, in, sdk]

- endpoints: e50
- what: PartyRoomService qua livekit-party-room.adapter: createRoom, mintJoinToken (grant theo role host/speaker/listener), updateParticipantPublish để nâng/hạ quyền publish, removeParticipant, deleteRoom, roomExists.
- why: Quyền nói trong Party Room là business state ở core-api (role, PARTY_MAX_SPEAKERS) nhưng chỉ SFU thực thi được việc cho/chặn publish audio, nên mỗi lần đổi role core-api phải đẩy grant xuống LiveKit.
- if missing: Listener có thể publish audio khi core-api không cho phép (hoặc speaker vừa bị hạ vẫn nói được); phòng đóng mà không deleteRoom thì SFU giữ room mồ côi.
- evidence: apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:21, apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:30
- review: sound — Đúng với adapter party: canPublish theo role, updateParticipantPublish sau commit, deleteRoom khi đóng phòng, roomExists cho sweeper.

### webhook trạng thái phòng gọi (c50) with calling [service, out, webhook]

- endpoints: e54
- what: LiveKit POST sự kiện participant_joined / participant_left / room_finished tới /api/v1/calling/webhooks/livekit (header Authorization là JWT ký bằng API key); CallingService.handleWebhookEvent xử lý idempotent.
- why: Chỉ SFU biết thời điểm hai bên thật sự nghe thấy nhau; core-api cần mốc đó để đặt startedAt, tính durationSeconds và giới hạn Voice Match, nên không thể chỉ dựa vào REST của client.
- if missing: Mất webhook thì CallSession không nhận mốc joined/startedAt đúng lúc; chỉ còn đối soát RoomService khi client đọc call làm backstop nên đồng hồ và chốt call trễ. Thiếu prefix /api/v1 thì LiveKit POST 404 âm thầm.
- evidence: apps/media-server/livekit.yaml:27, apps/core-api/src/modules/calling/webhooks/livekit-webhook.controller.ts:51
- review: corrected — Overstated: calling có đối soát RoomService (reconcileCallWithLivekit khi đọc call) làm backstop nên mất webhook không kẹt vĩnh viễn.

### webhook rớt kết nối phòng (c51) with party-room [service, out, webhook]

- endpoints: e55
- what: LiveKit POST participant_left / room_finished tới /api/v1/party/webhooks/livekit; PartyRoomService.handleWebhookEvent đánh dấu member/host disconnected hoặc đóng phòng.
- why: Host/member có thể rớt SFU mà không gọi REST leave; chỉ SFU thấy được nên core-api cần webhook để áp grace period reconnect của host và dọn membership.
- if missing: Phòng mà host mất mạng không bao giờ đóng/đổi trạng thái, danh sách member giữ người đã rớt; thiếu prefix /api/v1 thì webhook 404 âm thầm.
- evidence: apps/media-server/livekit.yaml:28, apps/core-api/src/modules/party-room/webhooks/party-livekit-webhook.controller.ts:51
- review: sound — handleParticipantLeft/markHostDisconnected đặt mốc grace; ifMissing khớp với sweeper chỉ bắt phòng rỗng hoặc SFU mất room.

### trạng thái cụm LiveKit (c52) with redis [infra, both, redis-state]

- endpoints: e50
- what: Khối redis.address trong livekit.yaml trỏ Redis (dev host.docker.internal:6379, k8s Redis managed ngoài cluster). Repo triển khai k8s với replicas: 1, strategy Recreate, hostNetwork: true.
- why: Theo docs/03 § 3.5 (mô tả cách LiveKit hoạt động), khi cấu hình Redis các node chia sẻ room data/message bus và chọn node còn tải để host room mới; một room phải nằm vừa trên một node. Repo chưa có test hay cấu hình nhiều node để kiểm chứng điều này.
- if missing: Không có Redis thì theo LiveKit/docs 03 § 3.5 các node không chia sẻ room state nên không dùng được multi-node; hiện manifest chỉ chạy 1 replica nên đường này chưa được kiểm chứng trong repo.
- evidence: apps/media-server/livekit.yaml:17
- review: corrected — Quy hành vi multi-node cho LiveKit và docs/03 § 3.5, nêu cấu hình thật của repo; bỏ các failure mode chưa kiểm chứng.

### client nối thẳng SFU (c53) with web [client, both, sdk]

- endpoints: e50, e51, e52
- what: Web (livekit-client) gọi room.connect(livekitUrl, accessToken) với token do core-api mint và URL region-aware; media đi thẳng trình duyệt <-> SFU trên 7880/7881/UDP, không qua core-api hay signaling-gateway.
- why: SFU chỉ tiết kiệm băng thông core-api nếu client nối trực tiếp; vì core-api không thấy media, bảo mật dựa hoàn toàn vào token TTL ngắn có grant theo role, và FE không bao giờ thấy LIVEKIT_API_SECRET.
- if missing: Nếu media phải đi qua core-api thì NestJS thành điểm nghẽn băng thông; nếu client không mở được UDP/7881 (firewall, hostNetwork thiếu) thì WebSocket vẫn connect nhưng audio câm.
- evidence: apps/web/src/shared/media/livekit.ts:27
- review: sound — room.connect(livekitUrl ?? env, accessToken) trong shared/media/livekit.ts; livekit.yaml ghi audio câm khi ICE candidate không route tới.

### chung cặp key/secret LiveKit (c54) with party-room [service, both, config]

- endpoints: e50, e54, e55
- what: Cặp LIVEKIT_API_KEY / LIVEKIT_API_SECRET của core-api phải khớp cặp key khai báo trong config LiveKit (dev inject qua LIVEKIT_KEYS, k8s ghi trong configmap) và key ký webhook (LIVEKIT_WEBHOOK_API_KEY).
- why: Core-api ký token và chữ ký server API bằng cặp này, còn LiveKit dùng đúng cặp đó để verify token và ký webhook; hai bên lệch thì token không verify được và WebhookReceiver ở core-api từ chối webhook (401).
- if missing: Mọi join token bị LiveKit từ chối và/hoặc mọi webhook bị core-api trả 401: không ai vào được phòng và core-api không nhận được trạng thái phòng.
- evidence: apps/core-api/src/common/livekit/livekit-sdk.client.ts:49, k8s/base/media-server/configmap.yaml:40
- review: sound — WebhookReceiver và RoomServiceClient cùng apiKey/apiSecret; compose inject LIVEKIT_KEYS và LIVEKIT_WEBHOOK_API_KEY từ LIVEKIT_API_KEY; configmap k8s ghi PHẢI khớp.

### scrape metrics LiveKit (c55) with external:prometheus [external, in, http]

- endpoints: e53
- what: Prometheus scrape cổng 6789 theo annotation prometheus.io/port trên pod; prometheus-alerts.yaml đặt cảnh báo dựa trên các metric này.
- why: Giới hạn một room/một node chỉ kiểm chứng được bằng số liệu tải thật; metrics phải ở cổng riêng để không lộ qua control port 7880 mà client truy cập.
- if missing: Không có số liệu CPU/băng thông/participant để quyết định nới PARTY_MAX_* hoặc thêm node; mất cảnh báo khi SFU quá tải.
- evidence: k8s/base/media-server/deployment.yaml:46
- review: sound — deployment.yaml có annotation prometheus.io/port 6789, prometheus_port 6789 ở yaml; prometheus-alerts.yaml tồn tại.

## Notes

- Không có dossier AST cho media-server: các entry kind port/webhook được thêm tay từ apps/media-server/livekit.yaml, nên 'endpoint' ở đây là cổng/webhook cấu hình chứ không phải route code.
- Một room phải vừa trên MỘT node (docs/03 § 3.5, ADR 0001): thêm replica chỉ scale số room, không chia một Party Room lớn; vì vậy PARTY_MAX_SPEAKERS/PARTY_MAX_MEMBERS là giới hạn an toàn bắt buộc. k8s/base/media-server/deployment.yaml ghim replicas: 1 và không có HPA.
- ADR 0005: pod media-server dùng hostNetwork: true (client kết nối thẳng IP Node trên UDP 50000-50200), tối đa 1 pod/Node. Service ClusterIP (k8s/base/media-server/service.yaml) chỉ phục vụ core-api gọi server API/webhook nội bộ, KHÔNG phải đường client vào RTC và không khai UDP range.
- Khác biệt dev vs k8s: dev dùng webhook http://host.docker.internal:3000 và redis host.docker.internal:6379 với use_external_ip: false; k8s configmap dùng DNS core-api.litmatch.svc.cluster.local:3000, Redis managed ngoài cluster (REPLACE_ME) và use_external_ip: true. Mọi webhook URL PHẢI có prefix /api/v1 (API_GLOBAL_PREFIX), thiếu thì LiveKit POST 404 âm thầm và CallSession kẹt pending.
- LiveKit v1.13.1 đọc key/secret từ chính file config (k8s configmap chỉ có placeholder) nên secret buộc nằm trong nội dung ConfigMap/mount lúc deploy; yêu cầu sinh ConfigMap từ template + secret ở bước deploy. Bài này không trích dẫn giá trị key nào.
- Multi-region: core-api chọn URL LiveKit trả cho client qua LIVEKIT_REGION_URLS (fallback LIVEKIT_URL); bất biến theo ADR 0005 là mọi URL trong map phải trỏ về cùng một cụm LiveKit (chung Redis), room được ghim endpoint theo region của host lúc tạo. Phần chọn URL nằm ở core-api, không ở media-server.
- Doc docs/services/realtime-gateway.md ghi rõ điều khiển LiveKit nằm NGOÀI phạm vi signaling-gateway: không có tương tác nào giữa signaling-gateway và media-server (signaling-gateway chỉ fanout event qua Redis, không mint token/gọi LiveKit).
- Chưa xác nhận bằng code: Redis của LiveKit có dùng chung instance với REDIS_URL của core-api hay không (dev trỏ host.docker.internal:6379 cùng Redis, k8s ghi 'giống' nhưng là placeholder).
- Webhook outbound của LiveKit được mô hình là entry kind port (method WEBHOOK-OUT) vì validator chỉ chấp nhận kind webhook khi khớp controller route của chính service; route nhận thật nằm ở core-api: POST /api/v1/calling/webhooks/livekit (calling) và POST /api/v1/party/webhooks/livekit (party-room), cả hai @Public + verify chữ ký trên rawBody.
