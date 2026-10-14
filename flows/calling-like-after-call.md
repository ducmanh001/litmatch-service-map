# Like sau cuộc gọi để thành bạn

id: flow:calling-like-after-call
group: Ghép đôi & gọi
services: module:calling, module:external:livekit, module:matching, module:friend, module:signaling-gateway

Sau Voice Match, mỗi bên bấm like; khi cả hai đã like thì Calling ghi reaction và nhờ Friend mở Friendship cùng Conversation trong cùng một giao dịch, nên không có friendship thiếu chat.

Calling đối soát call, khoá CallSession, ghi like rồi gọi Friend.ensureFriendship trong cùng giao dịch khi đối phương đã like.

## Participants

- web (client): Web client
- calling (service): Calling
- external:livekit (service): LiveKit
- matching (service): Matching
- friend (service): Friend
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> calling [call/http]: POST /calling/calls/:id/like (JWT)
2. calling -> postgres [call/shared-db]: getCall: đọc CallSession, kiểm tra là thành viên {apps/core-api/src/modules/calling/calling.service.ts:266}
3. calling -> external:livekit [call/sdk]: Đối soát: liệt kê participant của room (Pending có người trong room thì markJoined; Active mà đối phương đã rời thì endById (Completed). Lỗi LiveKit được nuốt và trả call nguyên trạng.) {apps/core-api/src/modules/calling/calling.service.ts:530}
4. calling -> matching [call/di-call]: Nếu call vừa bị end: chốt MatchSession voice (Chỉ chạy trong nhánh endById khi call không phải Friend và có matchSessionId.) {apps/core-api/src/modules/calling/calling.service.ts:688}
5. calling -> redis [async/redis-pubsub]: Nếu call vừa end: publish call.ended cho cả hai user {apps/core-api/src/modules/calling/calling.service.ts:710, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
6. redis -> signaling-gateway [async/redis-pubsub]: Gateway nhận realtime:user:* qua PSUBSCRIBE {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
7. signaling-gateway -> web [async/socket]: Emit call.ended tới room user:{userId} {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
8. calling -> postgres [call/shared-db]: Khoá CallSession, ghi VoiceMatchReaction, kiểm tra partner đã like {apps/core-api/src/modules/calling/calling.service.ts:356, apps/core-api/src/modules/calling/calling.service.ts:399}
9. calling -> friend [call/di-call]: Cả hai đã like: ensureFriendship(manager, nguồn VoiceMatch) {apps/core-api/src/modules/calling/calling.service.ts:404}
10. friend -> postgres [call/shared-db]: INSERT Friendship và Conversation (orIgnore) bằng manager chung {apps/core-api/src/modules/friend/friend.service.ts:120, apps/core-api/src/modules/friend/friend.service.ts:128}
11. calling -> friend [call/di-call]: areFriends: chỉ khi partner chưa like, để xác định matched (Code là `partnerLiked || await areFriends(...)`: nếu partner đã like thì matched=true và không gọi areFriends. areFriends đọc qua friendshipRepo, không dùng manager của giao dịch.) {apps/core-api/src/modules/calling/calling.service.ts:413}
12. calling -> web [return/http]: 200 {liked, matched, friendUserId}

## Invariants

- Reaction, Friendship và Conversation nằm cùng một giao dịch trên khoá pessimistic_write của CallSession, nên hai lượt like đồng thời được tuần tự hoá và không tạo Friendship thiếu Conversation.
- Calling không tự ghi Friendship: nó gọi FriendService.ensureFriendship với manager của mình để Friend là nơi duy nhất sở hữu bảng đó mà vẫn chung một giao dịch.
- Đối soát LiveKit chạy trước giao dịch like, nên call đã bị đối phương rời được chốt ended (kèm dọn Matching) trước khi ghi reaction.

## Failure modes

- when Redis publish call.ended lỗi: publishRealtimeEvent nuốt lỗi và chỉ log cảnh báo; client còn polling fallback, like vẫn xử lý bình thường.
- when LiveKit listParticipantIdentities lỗi: reconcileCallWithLivekit bắt lỗi, ghi debug và trả call nguyên trạng; like vẫn tiếp tục.
- when Vi phạm unique reaction (nhánh dự phòng khi race; hiếm vì đã khoá CallSession): Giao dịch rollback; likeCall bắt lỗi unique, đọc lại call và trả matched theo areFriends.
- when Call đang pending, hoặc là call kiểu Friend: Ném DomainException CALL_NOT_ENDED (409), không ghi reaction.

## Endpoints covered

- POST /calling/calls/:id/like [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
