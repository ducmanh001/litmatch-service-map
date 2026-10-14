# Từ hàng đợi ghép đôi đến cuộc gọi thoại

id: flow:matching-to-call
group: Ghép đôi & gọi
services: module:matching, module:economy, module:calling, module:signaling-gateway, module:media-server

Hai người xếp hàng ghép đôi, worker ghép cặp và hai bên xác nhận, sau đó vào cuộc gọi LiveKit. Matching sở hữu MatchSession, Calling sở hữu CallSession nên hai service phải gọi nhau qua DI và báo kết quả xuống client qua Redis và gateway.

Luồng một cặp người dùng đi từ vé ghép đôi, tới MatchSession, tới CallSession trên LiveKit và kết thúc cuộc gọi.

## Participants

- web (client): Web client
- matching (service): Matching
- economy (service): Economy
- calling (service): Calling
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway
- media-server (service): LiveKit SFU

## Steps

1. web -> matching [call/http]: POST /matching/tickets: xin vào hàng đợi ghép đôi
2. matching -> economy [call/di-call]: Trừ diamond cho lượt ghép thêm ngay trong giao dịch (Trước đó joinQueue kiểm tra hạn mức khách và VIP tier; việc trừ tiền chỉ xảy ra khi lượt ghép vượt hạn mức.) {apps/core-api/src/modules/matching/matching.service.ts:139}
3. matching -> postgres [call/shared-db]: Lưu MatchTicket cùng giao dịch với việc trừ diamond (Sau khi giao dịch commit, joinQueue gọi ensureEnqueued: ZADD NX vào hàng đợi Redis theo shard, SADD shard active rồi MatcherWakeup.notify để đánh thức worker; nếu bước Redis lỗi, ticket vẫn nằm trong DB và client retry cùng Idempotency-Key sẽ enqueue lại.) {apps/core-api/src/modules/matching/matching.service.ts:152, apps/core-api/src/modules/matching/matching.service.ts:618}
4. matching -> postgres [call/shared-db]: Worker ghép cặp: lưu MatchSession confirmed, ticket và thông báo (Trong tryPair, session được tạo thẳng với status Confirmed và cả hai ticket sang Confirmed; thông báo tạo bằng createWithManager nên cùng giao dịch. Worker chạy theo wake-up của MatcherWakeup (debounce) và có timer backstop định kỳ.) {apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:280}
5. matching -> redis [async/redis-pubsub]: Sau khi ghép, publish sự kiện realtime cho hai người (Worker cũng gọi sendPush của Notification sau giao dịch.) {apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:377}
6. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận qua PSUBSCRIBE realtime:user:* {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
7. signaling-gateway -> web [async/socket]: Relay sự kiện đã ghép được xuống socket của người dùng {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
8. web -> matching [call/http]: POST /matching/tickets/:id/confirm (đường tương thích pending_confirm) (Luồng mới đã confirmed ngay trong tryPair; endpoint này chỉ hoàn tất các ticket matched/session pending_confirm từ phiên bản cũ hoặc từ invite.)
9. matching -> postgres [call/shared-db]: Khoá session và ticket, đủ 2 bên mới lưu session, ticket và thông báo (Sau giao dịch, confirmTicket còn publish realtime và sendPush (best effort).) {apps/core-api/src/modules/matching/matching.service.ts:389}
10. web -> calling [call/http]: POST /calling/match-sessions/:matchSessionId/join
11. calling -> matching [call/di-call]: Đọc MatchSession qua MatchingService, Calling không tự ghi {apps/core-api/src/modules/calling/calling.service.ts:105}
12. calling -> postgres [call/shared-db]: Tạo CallSession gắn với matchSessionId (Sau đó mintJoinToken cấp token LiveKit và trả về cho client.) {apps/core-api/src/modules/calling/calling.service.ts:146}
13. media-server -> calling [call/webhook]: Webhook LiveKit báo người tham gia, Calling tìm CallSession theo roomName (Handler gọi markJoined, markDisconnected hoặc markRoomDisconnected, mỗi hàm mở một giao dịch.) {apps/core-api/src/modules/calling/calling.service.ts:492}
14. web -> calling [call/http]: POST /calling/calls/:id/end: kết thúc cuộc gọi
15. calling -> matching [call/di-call]: Báo Matching kết thúc phiên thoại của cuộc gọi (endById lưu CallSession trong giao dịch rồi cleanupEndedCall gọi endVoiceSessionForCall (ném lỗi nếu dữ liệu hỏng, không nuốt), xoá phòng LiveKit và publish realtime (hai việc này best-effort); sau đó endCall còn gọi endVoiceSession cho phiên của user.) {apps/core-api/src/modules/calling/calling.service.ts:688}
16. calling -> web [return/http]: Trả kết quả kết thúc cuộc gọi cho client

## Invariants

- Trừ diamond và lưu MatchTicket nằm chung một giao dịch trong joinQueue, nên không có ticket nếu việc trừ tiền thất bại và ngược lại.
- Matching là nơi ghi MatchSession; Calling chỉ đọc qua MatchingService.findSessionById và báo kết thúc qua endVoiceSessionForCall (trong cleanupEndedCall) hoặc endVoiceSession (trong endCall), giữ ranh giới sở hữu dữ liệu.
- Publish realtime và sendPush diễn ra sau giao dịch ghép cặp, nên thông báo chỉ đi khi session và ticket đã lưu xong.
- Đường confirm là đường tương thích: tryPair hiện tạo MatchSession ở trạng thái confirmed ngay, còn PendingConfirm chỉ giữ cho session cũ.

## Failure modes

- when Redis publish hoặc sendPush sau khi ghép cặp thất bại: Giao dịch đã commit nên MatchSession vẫn tồn tại; client có thể không nhận được sự kiện realtime tức thì.
- when Giao dịch tryPair hoặc confirmTicket lỗi giữa chừng: Session, thông báo và ticket được rollback cùng nhau, không để lại trạng thái nửa vời.

## Uncertain

- Gateway relay (bước 6-7) dùng chung bằng chứng từ VERIFIED FACTS.

## Endpoints covered

- POST /matching/tickets [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- MatcherWorkerService.onApplicationBootstrap [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- POST /matching/tickets/:id/confirm [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- POST /calling/match-sessions/:matchSessionId/join [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
- POST /calling/webhooks/livekit [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
- POST /calling/calls/:id/end [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
