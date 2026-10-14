# Chấp nhận lời mời và tạo phiên ghép

id: flow:matching-invite-accept
group: Ghép đôi & gọi
services: module:matching, module:safety, module:user, module:signaling-gateway

Invitee chấp nhận lời mời ghép: matching phải khoá invite, nhờ safety kiểm tra block/báo cáo, đọc trạng thái user, tạo 2 ticket cùng MatchSession đã confirmed trong một transaction rồi báo realtime cho cả hai bên qua Redis và signaling-gateway.

Accept lời mời: transaction precheck, transaction tạo ticket/session, rồi publish MatchConfirmed tới hai user.

## Participants

- web (client): Web client
- matching (service): Matching
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> matching [call/http]: POST /matching/invites/:id/accept (JWT) (Mọi route REST đều qua JwtAuthGuard toàn cục trừ khi @Public.) {apps/core-api/src/modules/matching/controllers/invite.controller.ts:93, apps/core-api/src/app/app.module.ts:154}
2. matching -> postgres [call/shared-db]: Transaction precheck: khoá invite, kiểm tra chủ lời mời, hết hạn, transition hợp lệ (Nếu invite đã Accepted thì trả lại session cũ (idempotent) và dừng ở đây.) {apps/core-api/src/modules/matching/services/invite.service.ts:447, apps/core-api/src/modules/matching/services/invite.service.ts:448}
3. matching -> safety [call/di-call]: canPair: kiểm tra block và báo cáo gần đây giữa hai user (MATCH_INTERACTION_POLICY được provide bằng useExisting: SafetyService. canPair dùng repository của Safety (không dùng manager của transaction) nên chạy trên kết nối riêng, trong lúc transaction precheck còn mở.) {apps/core-api/src/modules/matching/services/invite.service.ts:475, apps/core-api/src/modules/matching/matching.module.ts:70}
4. matching -> postgres [call/shared-db]: Nếu không ghép được: lưu invite Declined và COMMIT rồi mới báo lỗi 403 (Throw nằm ngoài transaction để phần Declined không bị rollback.) {apps/core-api/src/modules/matching/services/invite.service.ts:482}
5. matching -> postgres [call/shared-db]: Transaction chính: khoá lại invite, kiểm tra chủ lời mời, hết hạn, transition {apps/core-api/src/modules/matching/services/invite.service.ts:312, apps/core-api/src/modules/matching/services/invite.service.ts:313}
6. matching -> user [call/shared-db]: Đọc 2 User và yêu cầu cả hai ở trạng thái Active (Matching đọc entity User trực tiếp qua manager của transaction; không Active thì lỗi USER_BANNED (409).) {apps/core-api/src/modules/matching/services/invite.service.ts:336}
7. matching -> postgres [call/shared-db]: Insert 2 MatchTicket Matched (lỗi unique uq_match_tickets_active_user: 409) {apps/core-api/src/modules/matching/services/invite.service.ts:511}
8. matching -> postgres [call/shared-db]: Lưu MatchSession Confirmed, ticket Confirmed và invite Accepted {apps/core-api/src/modules/matching/services/invite.service.ts:363, apps/core-api/src/modules/matching/services/invite.service.ts:384}
9. matching -> redis [async/redis-pubsub]: Sau commit: publish MatchConfirmed {ticketId, sessionId} cho từng user (publishRealtimeEvent bắt lỗi publish và chỉ ghi log warn, nên lỗi không làm hỏng accept.) {apps/core-api/src/modules/matching/services/invite.service.ts:414, apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
10. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
11. signaling-gateway -> web [async/socket]: Emit MatchConfirmed tới room user:{userId} của hai bên {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
12. matching -> user [call/di-call]: Controller lấy hồ sơ inviter để dựng response {apps/core-api/src/modules/matching/controllers/invite.controller.ts:107}
13. matching -> web [return/http]: 200 OK: invite, session, inviteeTicketId và hồ sơ inviter

## Invariants

- Precheck chạy trong transaction riêng và commit trước khi throw, vì nếu throw trong cùng transaction thì trạng thái Declined vừa ghi sẽ bị rollback.
- Hai ticket, MatchSession và invite Accepted được ghi trong cùng một transaction nên hoặc có đủ hoặc không có gì; invite giữ Pending nếu một bên đang bận.
- Publish MatchConfirmed đặt sau khi transaction commit để client không nhận event về session chưa tồn tại trong DB.

## Failure modes

- when canPair trả false (có block hoặc báo cáo gần đây): Invite được lưu Declined và commit, request trả 403 INVITE_TARGET_UNAVAILABLE.
- when Một trong hai user đang ở queue hoặc phiên ghép khác (unique uq_match_tickets_active_user): Transaction rollback, trả 409 INVITE_ACCEPT_USER_BUSY, invite vẫn Pending.
- when Publish Redis thất bại: Lỗi chỉ được log warn, accept vẫn thành công; client còn polling fallback theo comment trong helper.
- when Người gọi không phải invitee: Trả 403 INVITE_FORBIDDEN, không ghi gì.

## Endpoints covered

- POST /matching/invites/:id/accept [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
