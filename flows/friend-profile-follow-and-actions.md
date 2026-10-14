# Theo dõi và xem hành động khả dụng trên hồ sơ

id: flow:friend-profile-follow-and-actions
group: Xã hội & nội dung
services: module:friend, module:user, module:safety

Khi người dùng mở hồ sơ người khác, module friend phải xác nhận hồ sơ tồn tại (user) và không bị block hai chiều (safety) trước khi trả trạng thái follow, hội thoại và ngưỡng first-chat, hoặc ghi/bỏ follow vào bảng của friend.

Ba endpoint GET actions, POST follow, DELETE follow của friend cùng gọi User để kiểm hồ sơ và Safety để kiểm block hai chiều trước khi đụng PostgreSQL.

## Participants

- web (client): Web client
- friend (service): Friend
- user (service): User
- safety (service): Safety
- postgres (infra): PostgreSQL

## Steps

1. web -> friend [call/http]: GET /profiles/:profileUserId/actions (JWT) để lấy trạng thái hành động
2. friend -> user [call/di-call]: Kiểm hồ sơ đích tồn tại (getByIdOrThrow) (assertTarget trước đó từ chối khi viewer trùng profile (400 SELF_PROFILE_ACTION).) {apps/core-api/src/modules/friend/services/profile-social.service.ts:398}
3. friend -> safety [call/di-call]: Kiểm block hai chiều: isBlocked(A,B) rồi isBlocked(B,A) (Nếu một chiều bị block, friend ném PROFILE_NOT_AVAILABLE (404 'Không tìm thấy hồ sơ').) {apps/core-api/src/modules/friend/services/profile-social.service.ts:546, apps/core-api/src/modules/friend/services/profile-social.service.ts:547}
4. friend -> postgres [call/shared-db]: Đọc follow, hội thoại 1-1 và đếm follower/following song song {apps/core-api/src/modules/friend/services/profile-social.service.ts:168, apps/core-api/src/modules/friend/services/profile-social.service.ts:585, apps/core-api/src/modules/friend/services/profile-social.service.ts:406}
5. friend -> postgres [call/shared-db]: Đếm first-chat trong ngày của profile để tính requiresGift (So sánh với PROFILE_DIRECT_MESSAGE_DAILY_FIRST_CHAT_THRESHOLD lấy từ config.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:608}
6. friend -> web [return/http]: Trả isFollowing, số follower/following, conversationId, requiresGift
7. web -> friend [call/http]: POST /profiles/:profileUserId/follow
8. friend -> user [call/di-call]: Kiểm hồ sơ đích tồn tại (getByIdOrThrow) {apps/core-api/src/modules/friend/services/profile-social.service.ts:398}
9. friend -> safety [call/di-call]: Kiểm block hai chiều trước khi cho follow {apps/core-api/src/modules/friend/services/profile-social.service.ts:546, apps/core-api/src/modules/friend/services/profile-social.service.ts:547}
10. friend -> postgres [call/shared-db]: Upsert ProfileFollow: active=true, lastFollowedAt=now() (Một câu INSERT ... ON CONFLICT (orUpdate) duy nhất, không mở transaction thủ công.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:92}
11. friend -> web [return/http]: Trả ProfileFollowDto (đang theo dõi = true)
12. web -> friend [call/http]: DELETE /profiles/:profileUserId/follow
13. friend -> user [call/di-call]: Kiểm hồ sơ đích tồn tại (getByIdOrThrow) {apps/core-api/src/modules/friend/services/profile-social.service.ts:398}
14. friend -> safety [call/di-call]: Kiểm block hai chiều trước khi bỏ follow {apps/core-api/src/modules/friend/services/profile-social.service.ts:546, apps/core-api/src/modules/friend/services/profile-social.service.ts:547}
15. friend -> postgres [call/shared-db]: Cập nhật ProfileFollow: active=false (không xóa dòng) {apps/core-api/src/modules/friend/services/profile-social.service.ts:116}
16. friend -> web [return/http]: Trả ProfileFollowDto (đang theo dõi = false)

## Invariants

- Cả ba endpoint đều gọi assertTarget (User) rồi assertNotBlocked (Safety) trước khi đọc/ghi bảng của friend, nên người bị block hai chiều không thể follow hay thấy trạng thái hành động.
- Block được kiểm cả hai chiều (A,B và B,A); lỗi trả 404 'Không tìm thấy hồ sơ' nên không lộ việc bị block.
- Friend chỉ sở hữu ProfileFollow, Conversation và ProfileChatContact; hồ sơ do User sở hữu và luật block do Safety sở hữu nên friend phải hỏi qua service chứ không tự đọc bảng của họ.

## Failure modes

- when Viewer thao tác với chính hồ sơ của mình: assertTarget ném SELF_PROFILE_ACTION (400), không chạm Safety hay DB follow.
- when Hồ sơ đích không tồn tại: UserService.getByIdOrThrow ném lỗi, request dừng trước bước kiểm block.
- when Một trong hai bên đã block bên kia: assertNotBlocked ném PROFILE_NOT_AVAILABLE (404), không ghi follow, không trả trạng thái.

## Endpoints covered

- GET /profiles/:profileUserId/actions [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- POST /profiles/:profileUserId/follow [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- DELETE /profiles/:profileUserId/follow [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
