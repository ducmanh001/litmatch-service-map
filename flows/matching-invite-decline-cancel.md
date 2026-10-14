# Từ chối hoặc huỷ lời mời

id: flow:matching-invite-decline-cancel
group: Ghép đôi & gọi
services: module:matching, module:user

Invitee từ chối hoặc inviter huỷ một MatchInvite. matching khoá dòng invite và kiểm tra hết hạn tại thời điểm thao tác, rồi gọi user để dựng DTO trả về.

Decline và cancel cùng một đường: khoá invite trong transaction, lazy-expire, lưu trạng thái, rồi lấy hồ sơ inviter từ user.

## Participants

- web (client): Web client
- matching (service): matching
- user (service): user
- postgres (infra): PostgreSQL

## Steps

1. web -> matching [call/http]: POST /matching/invites/:id/decline — invitee từ chối
2. matching -> postgres [call/shared-db]: Transaction: khoá dòng invite, kiểm tra chủ sở hữu, hết hạn, chuyển trạng thái (lockInvite ném 404 nếu không có; sai chủ sở hữu ném 403; assertNotExpired/assertTransition ném 409. Khi hết hạn chỉ gán status trong bộ nhớ rồi ném lỗi, transaction rollback nên trạng thái hết hạn không được lưu.) {apps/core-api/src/modules/matching/services/invite.service.ts:236, apps/core-api/src/modules/matching/services/invite.service.ts:605}
3. matching -> postgres [call/shared-db]: Lưu invite sang trạng thái Declined kèm respondedAt {apps/core-api/src/modules/matching/services/invite.service.ts:249}
4. matching -> user [call/di-call]: Lấy hồ sơ người mời để dựng MatchInviteDto {apps/core-api/src/modules/matching/controllers/invite.controller.ts:138}
5. matching -> web [return/http]: Trả MatchInviteDto (200 OK)
6. web -> matching [call/http]: POST /matching/invites/:id/cancel — inviter huỷ trước khi có phản hồi
7. matching -> postgres [call/shared-db]: Transaction: khoá dòng invite, kiểm tra chủ sở hữu, hết hạn, chuyển trạng thái (lockInvite ném 404 nếu không có; sai chủ sở hữu ném 403; assertNotExpired/assertTransition ném 409. Khi hết hạn chỉ gán status trong bộ nhớ rồi ném lỗi, transaction rollback nên trạng thái hết hạn không được lưu.) {apps/core-api/src/modules/matching/services/invite.service.ts:257, apps/core-api/src/modules/matching/services/invite.service.ts:605}
8. matching -> postgres [call/shared-db]: Lưu invite sang trạng thái Cancelled kèm respondedAt {apps/core-api/src/modules/matching/services/invite.service.ts:270}
9. matching -> user [call/di-call]: Lấy hồ sơ người mời để dựng MatchInviteDto {apps/core-api/src/modules/matching/controllers/invite.controller.ts:138}
10. matching -> web [return/http]: Trả MatchInviteDto (200 OK)

## Invariants

- Dòng invite được khoá pessimistic_write trước khi kiểm tra, nên hai thao tác đồng thời trên cùng invite (decline, cancel, accept) được tuần tự hoá.
- Lazy-expire kiểm tra expiresAt ngay lúc thao tác vì sweeper có thể chưa chạy; trạng thái Expired chỉ được lưu bởi sweeper, còn ở đây transaction bị rollback khi ném lỗi 409.
- Việc gọi user.getByIdOrThrow diễn ra sau transaction, ở controller, nên lỗi ở bước dựng DTO không làm hoàn tác trạng thái đã lưu.

## Failure modes

- when Invite không tồn tại: lockInvite ném INVITE_NOT_FOUND (404), transaction kết thúc.
- when Người gọi không phải invitee (decline) hoặc inviter (cancel): Ném INVITE_FORBIDDEN (403), không ghi gì.
- when Invite đang Pending nhưng đã quá expiresAt, hoặc chuyển trạng thái không hợp lệ: Ném INVITE_INVALID_TRANSITION (409); transaction rollback nên không lưu trạng thái Expired.
- when user.getByIdOrThrow thất bại sau khi đã lưu: Phản hồi lỗi nhưng trạng thái invite đã commit (Suy luận từ việc gọi nằm ngoài transaction).

## Endpoints covered

- POST /matching/invites/:id/decline [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- POST /matching/invites/:id/cancel [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
