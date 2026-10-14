# Kết thúc cuộc gọi và đồng hồ cuộc gọi

id: flow:calling-end-and-ticker
group: Ghép đôi & gọi
services: module:calling, module:matching, module:signaling-gateway, module:external:livekit

Một cuộc gọi Voice Match có thể kết thúc từ nhiều phía (poll GET, nút kết thúc, ticker nền). Calling chốt CallSession, nhưng MatchSession thuộc Matching, phòng thuộc LiveKit và báo realtime đi qua Redis tới gateway nên nhiều service phải phối hợp.

Mọi nhánh kết thúc đều qua CallingService.endById rồi cleanupEndedCall: Matching đóng session, xóa phòng LiveKit, publish call.ended qua Redis tới gateway.

## Participants

- web (client): Web client
- calling (service): Calling
- matching (service): Matching
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway
- external:livekit (service): LiveKit

## Steps

1. web -> calling [call/http]: GET /calling/calls/:id (poll trạng thái cuộc gọi) (Mọi route REST đều qua JwtAuthGuard toàn cục trừ @Public.)
2. calling -> postgres [call/shared-db]: Đọc CallSession theo id {apps/core-api/src/modules/calling/calling.service.ts:266}
3. calling -> external:livekit [call/sdk]: Đối soát: liệt kê participant đang có trong phòng (Nếu LiveKit lỗi, GET chỉ log debug và trả call hiện có, không trả 500.) {apps/core-api/src/modules/calling/calling.service.ts:530}
4. calling -> postgres [call/shared-db]: Pending: markJoined; Active mà đối phương vắng: endById khóa dòng, chốt ended (endById khóa pessimistic_write, chỉ lần chuyển trạng thái thật sự (justEnded) mới dọn dẹp và publish.) {apps/core-api/src/modules/calling/calling.service.ts:443}
5. calling -> matching [call/di-call]: cleanupEndedCall: endVoiceSessionForCall đóng MatchSession của cặp {apps/core-api/src/modules/calling/calling.service.ts:688}
6. matching -> postgres [call/shared-db]: Khóa match_sessions, đặt status Ended + endedAt (Bỏ qua nếu session không tồn tại, đã Ended hoặc không phải Confirmed; ném lỗi nếu cặp user không khớp.) {apps/core-api/src/modules/matching/matching.service.ts:588}
7. calling -> external:livekit [call/sdk]: Xóa phòng LiveKit, lỗi bị nuốt (phòng có thể đã tự đóng) {apps/core-api/src/modules/calling/calling.service.ts:693}
8. calling -> redis [async/redis-pubsub]: Publish call.ended cho cả hai user (publishRealtimeEvent bọc try/catch, lỗi chỉ log warn.) {apps/core-api/src/modules/calling/calling.service.ts:710, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
9. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
10. signaling-gateway -> web [async/socket]: Emit call.ended tới room user của từng người {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
11. calling -> web [return/http]: Trả CallSession (đã ended nếu vừa chốt)
12. web -> calling [call/http]: POST /calling/match-sessions/:matchSessionId/end (rời màn Voice Match)
13. calling -> matching [call/di-call]: findSessionById, kiểm user thuộc session (không thì 404) {apps/core-api/src/modules/calling/calling.service.ts:329}
14. calling -> postgres [call/shared-db]: Tìm CallSession theo matchSessionId; nếu có thì endById Completed (Sau endById chạy lại cleanupEndedCall như các bước 5-10: đóng Matching, xóa phòng, publish call.ended.) {apps/core-api/src/modules/calling/calling.service.ts:340}
15. calling -> matching [call/di-call]: endVoiceSession chốt MatchSession terminal cho cặp (idempotent) (Matching khóa match_sessions trong một transaction riêng; nếu đã Ended thì trả luôn session.) {apps/core-api/src/modules/calling/calling.service.ts:342, apps/core-api/src/modules/matching/matching.service.ts:502}
16. calling -> web [return/http]: Trả 200 sau khi call và MatchSession đã đóng
17. calling -> postgres [call/shared-db]: Ticker nền: quét pending quá hạn, active hết cửa sổ reconnect, active quá free-call (Chạy qua ManagedInterval theo CALLING_TICKER_INTERVAL_MS với clusterSingleton, mỗi tick tối đa 100 call mỗi loại.) {apps/core-api/src/modules/calling/jobs/call-ticker.service.ts:83}
18. calling -> postgres [call/shared-db]: Với mỗi call quá hạn: endById (pending_timeout, completed, free_limit) (Sau đó cleanupEndedCall như các bước 5-10. Lock FOR UPDATE chống race end-vs-tick.) {apps/core-api/src/modules/calling/jobs/call-ticker.service.ts:90}

## Invariants

- endById khóa CallSession rồi mới đổi trạng thái, nên end từ poll, nút bấm và ticker không thể chốt đôi và chỉ lần justEnded mới dọn phòng và publish.
- Matching đóng MatchSession trước khi publish call.ended, để user nhận event rồi tìm match lại không bị session confirmed cũ giữ chân.
- Calling không ghi match_sessions: chỉ Matching là nơi ghi, nên Calling phải gọi endVoiceSessionForCall/endVoiceSession.

## Failure modes

- when LiveKit RoomService lỗi khi GET /calling/calls/:id: reconcile bị bỏ qua, GET trả call hiện có; webhook và ticker là đường dự phòng.
- when deleteRoom thất bại (phòng đã tự đóng): Lỗi bị nuốt, chỉ log debug, luồng publish tiếp tục.
- when Publish Redis lỗi: Chỉ log warn, call vẫn đã ended trong DB; client còn polling dự phòng.
- when endVoiceSessionForCall ném lỗi (cặp user không khớp session): cleanupEndedCall dừng trước deleteRoom và publish, CallSession đã ended trong DB.
- when endById ném lỗi khi ticker xử lý call active quá free-call: Lỗi được catch và log theo từng call, thử lại ở tick sau. Với nhánh pending-timeout và reconnect-timeout không có catch từng call: lỗi thoát khỏi tick, ManagedInterval chỉ log lỗi và các bước sau của tick đó bị bỏ qua.

## Endpoints covered

- GET /calling/calls/:id [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
- POST /calling/match-sessions/:matchSessionId/end [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
- CallTickerService.onApplicationBootstrap [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
